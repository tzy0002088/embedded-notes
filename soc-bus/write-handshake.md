---
title: "总线写事务:完成信号与「完成」的三个层次"
date: 2026-08-15
tags: [soc, bus, axi, apb, ahb, posted-write, fence, store-buffer]
status: notes
---

## 背景

接 [mmio.md](./mmio.md) 的追问:core 执行 `sw` 写地址时,要不要等这个写动作完成?
总线会返回完成信号给 core 吗?如果不等,软件怎么知道写生效了?

## 提问

1. `sw` 写地址时,core 需要等待写完成吗?
2. 总线会返回完成信号吗?返回的"完成"意味着什么?

## 理解

### "完成"的三个层次

| 层次 | 含义 |
|------|------|
| ① 指令 retire | `sw` 从流水线里退休,架构上"执行完了" |
| ② 总线事务被接受 | slave 完成握手,数据被对方收下 |
| ③ 副作用真正发生 | 比如字节真的从串口线上移出去 |

### 普通内存写:posted write,不等

- 对可缓存区域,core 只保证 ①。store 进 **store buffer** 就算 retire,后面由总线接口单元异步发出去,即 **posted write(先斩后奏)**。
- 目的:性能。store 大多不在关键路径上,没必要让流水线停着等 DRAM。
- 正确性靠 store buffer 的转发机制 + 内存模型维持"好像已经写完"的幻觉。

### MMIO 写:必须真正到达设备

- PMA 把外设区域标记为**非幂等**:访问不可推测、不可缓存、不可合并。
- 实现上 MMIO 的 store 通常要真正走完总线握手(简单核如 Cortex-M 对 Device/Strongly-ordered 内存的写是非缓冲的,会真的停顿流水线等应答)。
- 原因:外设寄存器有副作用,不能丢、不能重发、不能乱序。

### 总线协议确实有完成信号

| 协议 | 完成信号 |
|------|---------|
| AXI | 写地址通道(AW)+ 写数据通道(W)之后,slave 在**写响应通道(B)** 回 `BRESP=OKAY` |
| APB | slave 拉高 `PREADY` 完成握手 |
| AHB | `HREADY` 类似 |

但注意:**这些应答只代表层次 ②**,而且高性能 core 里流水线不一定等过它——是互连/store buffer 在跟踪未完成事务,只有 store buffer 满了或遇到 `fence` 才阻塞。

### 最关键的坑:应答 ≠ 生效

协议应答的语义是"**slave 承诺收下这笔事务**",不是"副作用已完成"。

UART 例子:写 THR(发送保持寄存器)让 APB 握手完成,只意味着字节进了 UART 的发送寄存器(或 FIFO);此时移位寄存器还在按波特率逐 bit 往线上移(115200 波特下,一个字节约 87 µs)。**协议上早就"完成"了,实际发送还在后台进行。** 如果软件紧接着再写,会覆盖还没发完的 THR。

所以 xv6 的 `uartputc` 在写**之前**轮询状态寄存器:

```c
// xv6-riscv: kernel/uart.c
void uartputc(int c)
{
  ...
  // wait for Transmit Holding Empty to be set in LSR.
  while ((ReadReg(LSR) & LSR_TX_IDLE) == 0)
    ;
  WriteReg(THR, c);   // 此时上一字节已进入移位寄存器,THR 空出来
}
```

驱动软件永远是"**问状态寄存器 / 等中断**"来确认副作用完成,而不是相信总线应答。

### 顺序问题:fence

总线应答保证"这一笔到了",但不保证"我前一笔到了"。RISC-V 内存模型里 store 和后续不同地址的 load 之间没有默认顺序。所以 xv6 的 virtio 驱动在写通知寄存器前要插 fence:

```c
// xv6-riscv: kernel/virtio_disk.c
  __atomic_thread_fence(__ATOMIC_SEQ_CST); // 编译成 fence rw,rw
  *R(VIRTIO_MMIO_QUEUE_NOTIFY) = 0;        // 保证前面的 desc 写对设备可见,再踢一脚
```

### 时间线小结

```
sw 0x10000000, t0
   │
   ├─① 指令 retire:瞬间(若在 store buffer 内)
   │
   ├─② 总线握手:AXI 的 BRESP / APB 的 PREADY
   │      ↳ 语义:"UART 收下了这个字节",写响应回来了
   │      ↳ core 大概率没在这等,是互连在跟踪
   │
   └─③ 副作用完成:87µs 后字节才从线上发完
          ↳ 没有任何硬件信号通知 core 这件事
          ↳ 软件自己去读 LSR 或等中断
```

## 要点

- 普通内存 store:posted write,进 store buffer 即 retire,不等;MMIO store:走完握手才继续
- 总线**有**完成信号(AXI `BRESP` / APB `PREADY` / AHB `HREADY`),但语义只是"slave 收下"
- 副作用完成(层次 ③)没有任何硬件通知,软件必须主动问:状态寄存器轮询或中断
- 跨设备顺序靠 `fence`(xv6 里 `__atomic_thread_fence(__ATOMIC_SEQ_CST)`)

## 参考

- xv6-riscv 源码:`/home/tzy/inspiration/xv6-riscv/kernel/uart.c`(uartputc)、`kernel/virtio_disk.c`(QUEUE_NOTIFY 前的 fence)
- AMBA AXI / APB / AHB 规范中的写响应与握手信号
- 延伸:[mmio.md](./mmio.md) — MMIO 机制
