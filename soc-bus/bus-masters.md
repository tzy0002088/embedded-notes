---
title: "DMA master 与多 master 世界:双端口 IP、仲裁、一致性根源"
date: 2026-08-15
tags: [soc, bus, dma, usb, arbitration, coherency]
status: notes
---

## 背景

理解了 core 与总线的通信之后,追问:USB 控制器内的 DMA master、通用 DMA master
是怎么和总线通信的?——总线上不止 core 一个 master。

## 提问

1. USB 控制器内的 DMA master / 通用 DMA master 是怎么和总线通信的?
2. 多个 master 共存会带来什么?

## 理解

### DMA master 与 core 地位平等

DMA master 和 core 在总线上的地位完全平等,都是 master。链路结构一模一样:
把 core 换成 DMA 引擎即可。区别只有一件事:**谁在指挥它发起事务**——
core 被指令流指挥,DMA 被寄存器配置 + 触发指挥。

### 双端口 IP:一个 IP,对总线既当 slave 又当 master

USB 控制器里嵌的 DMA 和通用 DMA 控制器,结构上都是双端口:

```text
                    总线(AHB / AXI)
              ▲                      │
         slave 端口              master 端口
      (CPU 配寄存器)           (DMA 引擎搬运数据)
        ┌─────┴────────────────────┴─────┐
        │  USB 控制器(内含 DMA 引擎)       │
        │                                │
        │  slave 侧:控制寄存器文件         │
        │  (端点配置、DMA 地址、长度…)      │
        │                                │
        │  master 侧:DMA 状态机           │
        │  (读 FIFO → 写内存 / 内存 → FIFO)│
        └────────────────────────────────┘
```

- **slave 端口**:CPU 通过它写配置(源地址、目的地址、长度、使能)——普通 MMIO 写路径
- **master 端口**:DMA 引擎发起真正的数据搬运——USB 控制器把端点 FIFO 数据直接写进内存,
  或从内存读数据填 FIFO,**全程不经 CPU**

互连译码器眼里这是两个不同的端口。这和 [[fmc-sdram]] 里"同一个 FMC 两种语义"同构:
同一个 IP 按端口扮演不同角色。

### 一次搬运在总线上的样子

CPU 配好 DMA 后,memory-to-memory 搬运就是一个循环状态机:

```text
CPU 写 DMA 寄存器: 源=0x20001000, 目的=0x20003000, 长度=1024
      │ 触发(软件触发或外设请求)
      ▼
DMA 引擎循环:
  ① 发读事务:  读 0x20001000(AR/R 通道,或 AHB 读)
  ② 数据暂存进 DMA 内部 FIFO
  ③ 发写事务:  写 0x20003000(AW/W/B 通道,或 AHB 写)
  ④ 地址 +4,计数 -1,回到 ①
  ⑤ 搬完 → 置状态位 / 发中断给 CPU
```

一个 DMA 同时扮演两种角色:对源地址它是读的 master,对目的地址它是写的 master。

### 多 master 世界的两个直接后果

**① 互连要做多对多仲裁。** 反压链变成矩阵式的:

```text
STM32F4 的总线矩阵:
masters:  ICode │ DCode │ S-bus │ DMA1 │ DMA2 │ ETH MAC │ USB OTG HS
              │      │       │       │      │        │
              ▼      ▼       ▼       ▼      ▼        ▼
           ┌─────────────────────────────────────────┐
           │         总线矩阵(仲裁 + 译码)             │
           └──────┬──────┬──────┬──────┬─────────────┘
                  ▼      ▼      ▼      ▼
slaves:      Flash   SRAM1  SRAM2   FMC │ AHB 外设 │ APB1/2 桥
```

两个 master 同时访问同一个 slave 时,矩阵按优先级仲裁,输的那方被反压。
这也就是 AHB-Lite 只支持单 master、多 master 必须用总线矩阵的原因。

**② 一致性问题的根源。** 之前说的"DMA 来读 buffer 前要 clean cache / 排空 store buffer",
本质就是:core 和 DMA 是两个平等的 master,**谁都不知道对方私有缓冲(cache、store buffer)
里有什么**。core 认为"写完了"的数据可能还躺在 core 的 cache 里,总线上的 DMA 看到的还是
旧内存。多 master + 各管各的私有状态 = 一致性难题,软硬件协同(snoop、cache clean、fence)
都是在给这件事擦屁股。

### 对比表

| | core | DMA master |
|---|---|---|
| 事务来源 | 指令流(load/store) | 寄存器配置 + 触发信号 |
| 谁指挥 | 当前程序 | CPU 预先写的配置 + 硬件请求 |
| 数据通路 | 经过 core(cache/寄存器) | 不经 core,设备↔内存直连 |
| 对总线的语言 | 内部方言 + 协议桥 | 通常原生说 AXI/AHB |

最后一个区别很能说明问题:core 是"外来的",所以需要翻译官(协议桥);
DMA 控制器**生来就是总线上的 IP**,自己就讲 AXI/AHB,不需要桥——这正是"IP 生态复用"
的直接体现。

## 要点

- DMA master 与 core 平级;区别只在指挥者:指令流 vs 寄存器配置 + 触发
- 外设控制器常是双端口 IP:slave 端口收配置(MMIO),master 端口搬数据
- 一次搬运 = 循环的"读事务 → 内部 FIFO → 写事务"
- 多 master 需要互连仲裁(总线矩阵);一致性问题的根源是多 master + 各自私有缓冲
- DMA 控制器原生说总线协议,不需要协议桥

## 参考

- [bus-protocols.md](./bus-protocols.md) — 协议桥与三种总线分工
- [fmc-sdram.md](./fmc-sdram.md) — 同一个 IP 两种语义的另一个实例
- [glossary.md](./glossary.md) — 术语表
