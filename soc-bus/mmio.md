---
title: "MMIO:RISC-V core 如何通过寄存器与外设打交道"
date: 2026-08-15
tags: [soc, mmio, riscv, bus, pma, volatile]
status: notes
---

## 背景

开始学习 SoC/MCU 总线级知识时产生的根本疑问:core 本身不知道某个地址背后是外设还是内存,
那它怎么通过寄存器与外设打交道?读 xv6 源码时看到 UART 直接用 volatile 指针访问
`0x10000000`,想搞清楚背后的硬件机制。

## 提问

1. core 不知道某个地址是外设,那 load/store 是怎么到达外设的?
2. 软件是怎么知道外设在哪个地址的?
3. RISC-V 访问外设和 x86 有什么不同?

## 理解

### 核心观点:core 不需要"知道"

CPU 只负责发出带地址的 load/store 请求,从不关心地址背后是什么。"这个地址是内存还是外设"
由**总线互连的地址译码器**决定,由**外设自己解释**这次读写。core 不参与判断。

### MMIO:外设寄存器被塞进统一地址空间

SoC 把外设寄存器映射到和内存同一个地址空间(部分地址给 DRAM,部分给外设)。例如 qemu virt 平台:

```
0x80000000 起  → DRAM
0x10000000 起  → UART0
0x10001000 起  → virtio MMIO
```

### 硬件侧:地址译码器负责路由

CPU 的 load/store 经 LSU → 总线互连(AXI interconnect,或更老的 AHB/APB),互连里的地址译码逻辑:

- 根据地址高位判断请求落在哪个 slave 的地址窗口
- 把事务路由给对应 slave(DRAM 控制器、UART、GPIO……)

地址映射是**芯片设计时定死的 SoC memory map**,体现在译码器电路里,不在 CPU 里。

外设端(以 APB 为例)看到的是一组信号:`PSEL`(被选中)、`PADDR`、`PWDATA`、`PWRITE`。
当 `PSEL` 有效且 `PADDR` 匹配自己的某个寄存器时,锁存数据并**触发动作**——如 UART 的 TX 寄存器被写就启动发送逻辑。

**写寄存器的本质是"送消息"**:数据值不重要,写这个动作本身有副作用。读寄存器同理,读到的是外设当前状态(如 RX 队列),而不是某个"存储单元"。

### RISC-V 这一侧的特点

- **没有 x86 那样的独立 IO 指令**(`in`/`out` 和 IO 端口空间),访问外设就是普通 load/store,和访问内存同一条路径。
- **PMA(物理内存属性)** 标记 MMIO 区域:**非缓存(non-cacheable)、不可推测(non-speculative)**。否则硬件可能把外设寄存器读放进 cache 拿到过期值,或预取触发副作用(如 FIFO 弹出数据)。由 SoC 的 PMA 检查器保证。
- 外设访问之间需要 **`fence`** 保证顺序和可见性(MMIO 是非缓存区,普通 load/store 可能被重排或合并)。

### 软件侧:"知道地址"的是软件

软件从 SoC 手册、linker script 或设备树(OpenSBI 传 DTB 给内核)拿到地址映射,然后就是宏定义 + volatile 指针。xv6 里最直接的例子:

```c
// xv6-riscv: kernel/uart.c
#define UART0 0x10000000L
#define THR   0  // transmit holding register

#define Reg(reg) ((volatile unsigned char *)(UART0 + reg))
#define WriteReg(reg, v) (*(Reg(reg)) = (v))

void uartputc(int c)
{
  ...
  WriteReg(THR, c);   // 就是一次普通 sb 指令,写到 0x10000000
}
```

编译器生成的就是一条普通 `sb`。CPU 执行时完全不知道这是 UART;但这条 store 在总线上被译码、路由到 UART 控制器,UART 硬件收到写就启动发送。

**"约定"存在于**:芯片设计者定地址映射 → 驱动代码用同样的地址去访问 → 硬件按约定解释。

### 常见追问

- **为什么需要 `volatile`?** 编译器不知道 0x10000000 是寄存器,没有 volatile 可能把重复读写优化掉(轮询只读一次、合并两次写)。外设寄存器每次访问都有意义,必须用 volatile 阻止优化。注意 volatile 只管编译器,不管硬件重排——硬件顺序靠 fence。
- **读同一地址可能每次结果不同**——因为那不是存储,是硬件实时状态。这也是 MMIO 不能进 cache 的原因:cache 假设同一地址内容稳定。
- **内核为什么用设备树而不是硬编码 MMIO 地址?** 同一内核跑在不同板子上,外设布局不同。DTB 把"约定"从硬编码变成启动时告知,驱动的 probe 据此 ioremap(有 MMU 时把物理地址映射到虚拟地址才能访问)。

## 要点

- core 只发"带地址的读写",不负责"理解";路由和解释是互连和外设的事
- 一条链:`sw` 指令 → 地址落入外设窗口 → 总线译码路由 → 外设锁存数据并触发副作用 → 软件读回状态
- MMIO 语义:写 = 送消息(有副作用),读 = 问状态;与普通内存"读写存储"完全不同
- RISC-V 无独立 IO 指令,全部 load/store;PMA 保证 MMIO 不缓存、不推测;fence 保证顺序
- 软件侧:volatile 指针 + 手册/DTB 的地址约定

## 参考

- xv6-riscv 源码:`/home/tzy/inspiration/xv6-riscv/kernel/uart.c`、`kernel/memlayout.h`
- RISC-V 特权规范 §3.6 Physical Memory Attributes
- 延伸:[write-handshake.md](./write-handshake.md) — 写事务的完成语义
