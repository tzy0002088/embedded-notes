---
title: "协议桥:core 内部方言如何翻译成 AXI/AHB/APB 事务"
date: 2026-08-15
tags: [soc, bus, axi, ahb, apb, tilelink, protocol-bridge]
status: notes
---

## 背景

前面搞清了 core 内部执行 sw 的过程和各种握手,但 core 本身并不输出 AXI/AHB/APB 信号——
那 RISC core 和这三种总线之间到底是怎么通信的?

## 提问

1. core 与 AXI/AHB/APB 之间是怎么通信的?
2. 为什么一个 SoC 里需要三种总线,它们怎么分工?

## 理解

### 核心事实:core 不说总线协议,它说"内部方言"

AXI/AHB/APB 是**总线协议**;core 内部有自己的接口(LSU 发出的一组信号),
中间靠**协议桥/适配器**翻译。通信本质不变:valid/ready 式握手,桥只是状态机,
把一种握手风格映射成另一种。

```text
┌─────────────── core ───────────────┐
│  LSU(load/store unit)              │
│                                    │
│  mem_valid   ──▶ "我有请求"         │  ← core 的"内部方言"
│  mem_addr    ──▶ 地址               │    (valid/ready 风格)
│  mem_wdata   ──▶ 写数据             │
│  mem_wstrb   ──▶ 字节选通           │
│  mem_ready   ◀── "你接下了吗"       │
│  mem_rdata   ◀── 读数据             │
└──────────────┬─────────────────────┘
               ▼
      ┌──────────────────┐
      │ 协议桥(状态机)     │  ← 方言翻译成标准总线协议
      │ 内部接口 ↔ AXI    │
      └────────┬─────────┘
               ▼ AXI(5 通道)
      ┌────────────────────┐
      │ AXI 互连(地址译码)  │
      └───┬────────────┬───┘
          ▼            ▼
     ┌─────────┐  ┌─────────────┐
     │ DDR 控制器│  │ AXI→APB 桥  │
     └─────────┘  └──────┬──────┘
                         ▼ APB
                   ┌────────────┐
                   │ UART/GPIO… │
                   └────────────┘
```

I-cache miss 取指也走同一条路:取指只是"另一个 master 发起的读事务"(AR/R 通道),
和数据 load 没有区别。

### 一条指令怎么变成总线事务

```text
load → 内部 {valid, addr}  →  桥 →  AR 通道(读地址,ARVALID/ARREADY 握手)
                           →  等 R 通道(数据回来,RVALID/RREADY/RLAST/RRESP)
                           →  数据送回 mem_rdata,拉高 mem_ready

store → 内部 {valid, addr, wdata, wstrb}
                           →  桥 →  AW 通道(写地址)
                           →  W 通道(写数据)
                           →  等 B 通道(BRESP 响应)
                           →  拉高 mem_ready,core 侧完成
```

握手映射的难易程度:

- **内部 valid/ready ↔ AXI VALID/READY**:几乎直连,桥很薄(RISC-V 小核的 AXI-Lite adapter
  就几百行逻辑)
- **↔ AHB**:桥要产生地址相、并在 HREADY 低时保持所有信号,多一个状态
- **↔ APB**:桥要产生两相时序(PSEL 拉高 = setup 相,再拉 PENABLE = access 相,等 PREADY 完成)

### 三个协议在 SoC 里的分工

协议是"按带宽和面积挑的":

| | AXI | AHB | APB |
|---|---|---|---|
| 定位 | 高性能主干 | 中速设备 | 慢速外设 |
| 通道 | 5 个独立通道(读写分离) | 单地址/数据总线,流水线化 | 无流水,两相时序 |
| burst / 乱序 / outstanding | 全支持 | 部分支持 | 不支持 |
| 面积功耗 | 大 | 中 | 小 |
| 典型用途 | core↔互连↔DDR、DMA | ARM 生态中速外设 | UART/GPIO/I2C/定时器 |

一个 SoC 里它们**同时存在、串成树**:AXI 是主干,APB 是叶子,中间架桥(AXI→APB 或 AHB→APB)。
FMC 反压传导(见 [[fmc-sdram]]:UART 的 PREADY → 桥 → HREADY → core 流水线冻结)
就是桥在做协议转换时把反压一级级传回去。

### 真实芯片里的对应物

- **Rocket/SiFive 系(RISC-V)**:core 内部说 **TileLink**(带缓存一致性语义的协议),
  出来过一个 TL→AXI 转换器接 AXI 互连,外设区再 AXI→APB。所以 Rocket 的 `sw` 走了两座桥。
- **PicoRV32 这类小核**:内部就是 `mem_valid/mem_ready` 四件套,官方提供
  `picorv32_axi`/`picorv32_axilite` adapter,拿到 AXI 世界里接任何现成 IP。
- **Cortex-M**:core 直出 **AHB-Lite**(AHB 为单 master 场景的精简版),
  后面 AHB→APB 桥到外设——就是 STM32F1 那套。

### 为什么不让 core 直接说 AXI?

1. **解耦**:core 微架构(流水线深度、乱序、缓存层次)和总线协议可以独立演进,换个总线只换桥
2. **生态复用**:AXI/APB 是通用标准,现成 IP(DDR 控制器、DMA、UART)全说这个;
  core 厂商配个桥就能接入整个 IP 生态
3. **一致性扩展**:core 内部协议(TileLink、ACE)带缓存一致性语义,AXI 本身不带,
  桥正好做语义边界

## 要点

- core 与总线之间永远隔着"翻译官"(协议桥):core 说内部握手,桥变成标准总线事务
- 内部 valid/ready ↔ AXI 几乎直连,↔ AHB/APB 需要状态机产生地址相/两相时序
- AXI 主干、APB 叶子、中间架桥,协议选择取决于"这条总线服务什么设备"
- 取指和数据访问共用同一个总线接口,取指就是一次读事务
- core 始终只会 load/store,协议和拓扑是 SoC 集成的事

## 参考

- [sw-to-bus.md](./sw-to-bus.md) — store 数据通路与三种芯片配置
- [bus-masters.md](./bus-masters.md) — DMA master 与多 master 世界
- [glossary.md](./glossary.md) — 术语表
- AMBA AXI / AHB / APB 规范
