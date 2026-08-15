---
title: "FMC SDRAM:总线知识的第一个具体硬件实例"
date: 2026-08-15
tags: [soc, stm32, fmc, sdram, ahb, dma, cache]
status: notes
---

## 背景

学习 core 与总线交互时,拿 STM32 FMC(Flexible Memory Controller)接口的 SDRAM 作为具体实例,
把 posted write、反压、波形翻译这些抽象概念落在一个真实硬件上。

## 提问

1. FMC 接口的 SDRAM,core 读写时会等这笔 load/store 在硬件层面完成吗?
2. 写 FMC memory map 的地址,是怎么触发 FMC 产生波形给 SDRAM 的?

## 理解

### load 会等,store 默认不等

FMC SDRAM 在 core 眼里是 **Normal 内存**(不是 MMIO),走 posted write 语义:

- **load:必须等**。FMC + SDRAM 的延迟(CAS latency、行激活、FMC 时序寄存器配的 wait state)
  直接体现为总线上 HREADY 拉低,流水线在 load-use 处停住。这也解释了 F4 从 SDRAM 跑代码明显变慢:
  Cortex-M4 核本身无 cache,F4 的 ART 加速器只针对内部 Flash(约 1KB 指令缓冲 + 128B 数据缓冲),
  管不到 0xC0000000 的 SDRAM,所以每条取指都走 S-bus → FMC → SDRAM 付完整延迟
  (SDRAM 时钟 = HCLK/2,一次随机读折合十几到几十个 core 周期,分支密集更糟)。
  精确地说不是无条件等:后续指令不依赖这个结果时,流水线继续跑,
  读事务在总线上挂着;带 cache 的型号(F7/H7 用 MPU 配成可缓存)命中时根本不产生总线事务。
- **store:默认不等,两级缓冲**:

```text
core 写缓冲(write buffer)  →  总线  →  FMC 写 FIFO  →  SDRAM
   ① 指令 retire,瞬间           ② 握手,很快     ③ 按 SDRAM 时序慢慢发
```

  单条 store 只走到 ① 就退休。只有**缓冲被反压**时才停:写循环太快 → FMC 写 FIFO 满 →
  HREADY 拉低 → 总线停 → core 写缓冲也满 → core 在下一条 store 处停住。
  所以连续写循环吞吐 = SDRAM 写带宽,但每一条 store 指令本身不等。
- 为什么能这么干:SDRAM 是纯数据存储、无副作用,写只是"让某地址变成某值"——和 UART 的
  TX 寄存器"写一次就触发一次发送"本质不同。

### 什么时候必须让 store"真正落地"

posted write 的代价:core 认为写完了,不代表别人看得见。需要显式同步的场景:

- **DMA 接着读这块 buffer**:启动 DMA 前 `__DMB()`/`__DSB()` 把 core 写缓冲排空;
  带 D-cache 的型号(H7)还得先 clean D-cache,否则 DMA 从 SDRAM 读到旧数据
- **把代码写到 SDRAM 再执行**(bootloader 搬程序):M7 上要 invalidate I-cache
- 单核自己写自己读没问题:写缓冲有转发,后续读同一地址会拿到新值

### 写 FMC 数据窗口地址 = FMC 状态机产生 SDRAM 波形

core 写 0xC0000000 区域,对 core 只是一次普通 store;FMC 这个 slave 收到 AHB 事务后,
**用自己的状态机把它翻译成 SDRAM 总线上的信号序列**。core 全程不知情。

```text
core: str r1, [r2]          (r2 = 0xC0001234)
        │
        ▼
AHB 事务: HADDR=0xC0001234, HWDATA=..., HWRITE=1
        │
        ▼
FMC 译码: SDCR 配置决定地址切分
   0xC0001234 → bank 选择(SDNE0/1) + 行地址 + 列地址
        │
        ▼
SDRAM 状态机产生波形序列
```

地址切分规则在 `SDCR`/`SDTR` 里配(row 位数、column 位数、bank 位),
如 13 行 9 列时把 AHB 地址按 bit 切成 [bank | row[12:0] | col[8:0]]。

### SDRAM 波形:四根控制线的命令编码

命令是 **CS#/RAS#/CAS#/WE# 四根线的电平组合**,配合 SDCLK 时钟沿输出:

| 命令 | CS# | RAS# | CAS# | WE# | A[12:0] 上的内容 |
|------|-----|------|------|-----|-----------------|
| ACT(激活行) | 0 | 0 | 1 | 1 | 行地址 |
| READ | 0 | 1 | 0 | 1 | 列地址 |
| WRITE | 0 | 1 | 0 | 0 | 列地址 |
| PRE(预充电) | 0 | 0 | 1 | 0 | 模式位 |
| REF(刷新) | 0 | 0 | 0 | 1 | — |

一次写的典型时序:

```text
SDCLK   _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
CMD      ACT    NOP    WRITE   NOP    NOP   ...
A[12:0]  行地址   x     列地址    x      x
DQ       xxxxx  xxxxx   D0     D1    D2 ...
                  ↑ tRCD(行激活到写命令的间隔,SDTR 配置)
```

- 状态机先发 **ACT** 打开行(把整行从电容阵列读到行缓冲/灵敏放大器)
- 等 `tRCD` 个周期(SDTR 里配)
- 发 **WRITE** + 列地址,数据在 DQ 线上逐拍送出,DQM 做字节掩码
- 可选自动预充电把行写回电容(时序全按 SDTR)
- FMC 内部有刷新计数器,按 SDTR 的 COUNT 周期性插入 **REF**,和读写交替进行——
  所以波形里会看到随机插入的刷新周期

读路径同理:ACT → 等 tRCD → READ → 等 CAS latency(CL)→ DQ 上数据回来 → FMC 把 HREADY 拉高
完成 AHB 读事务。**"load 要等"在波形层面的样子:数据没从 DQ 回来,HREADY 一直是低。**

### 同一个 FMC,两种语义

| FMC 区域 | 语义 | 行为 |
|---------|------|------|
| FMC 配置寄存器区 | Device(强序) | 写时序参数、发初始化命令严格按序、真到达 → 初始化序列可靠 |
| SDRAM 数据窗口 | Normal 内存 | 可缓冲、可缓存,store 走 posted |

写数据窗口(0xC0000000)→ 状态机产生 SDRAM 波形;写配置寄存器 → 只改控制器内部参数,
**不产生任何 SDRAM 波形**。同一个控制器按地址区间被区别对待——"地址决定语义",
就是 RISC-V PMA 在 Cortex-M 上的对应物(ARM 默认内存映射 + MPU 覆盖)。

posted write 的"层次③"(见 [[write-handshake]])在这里具象化:store 到 Normal 内存不等落地,
而"落地"的物理形式就是这些波形——SDRAM 电容充上电、数据真的存进阵列。

## 要点

- load 等(数据依赖),store 不等(两级缓冲 posted);只有缓冲被反压时才停
- 连续写循环吞吐由 SDRAM 写带宽决定,单条 store 很快
- FMC = 总线事务 → SDRAM 协议的翻译器,命令 = CS#/RAS#/CAS#/WE# 电平组合
- 所有时序参数来自 SDCR/SDTR 配置寄存器,所以初始化序列必须强序(Device 语义)
- 同一控制器两种语义:寄存器区强序、数据窗口 posted,靠地址区分
- DMA/自修改代码场景:store 需要主动同步(DMB/DSB + cache clean/invalidate)

## 参考

- ST RM0090(F429)FMC 章节
- [write-handshake.md](./write-handshake.md) — "完成"的三个层次
- [sw-to-bus.md](./sw-to-bus.md) — core 与总线的交互链路
