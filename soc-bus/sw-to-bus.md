---
title: "从 sw 指令到总线交易:core 与总线的交互全链路"
date: 2026-08-15
tags: [soc, bus, axi, ahb, apb, pipeline, cache, riscv, store-buffer]
status: notes
---

## 背景

追问"core 执行 sw 时,core 与总线之间的交互是什么、什么过程",按三种芯片配置分别拆解:
① 无 cache 无 store buffer(MCU 级,如 STM32F1 的 Cortex-M3)
② 有 cache、无乱序、无 store buffer(RV32)
③ 有 cache 有 store buffer(高性能核)

## 提问

1. core 发起 sw,core 与总线之间的交互是什么?
2. 不同芯片配置下,这条链路有什么差异?

## 理解

### 配置①:MCU 级(无 cache,无 store buffer)

交互的物理形态:**一组 AHB 信号 + 一根 HREADY,一拍一拍握手**。store 不完成,指令不退休。

```text
         ┌─────────────── Cortex-M3 ───────────────┐
         │                                          │
         │  HADDR   ──────────────▶ 地址             │
         │  HWDATA  ──────────────▶ 写数据           │
         │  HWRITE  ──────────────▶ 1=写             │
         │  HSIZE   ──────────────▶ 字节/半字/字      │
         │  HTRANS  ──────────────▶ 传输类型(NONSEQ) │
         │  HREADY  ◀────────────── slave 说"可继续" │
         │  HRDATA  ◀────────────── (写用不到)        │
         └─────────────────┬────────────────────────┘
                           ▼
                    AHB 总线矩阵(地址译码)
```

store 的执行只做两件事:**把信号驱动出去,然后每个时钟沿采样 HREADY**。
HREADY 为高 → 交易完成,指令退休;为低 → 所有信号原地保持,**整条流水线冻结**。

走一遍 STM32F1 上 `str r1, [r0]`(r0 = 0x40011004,USART1->DR):

```text
IF: 从 flash 取指(本身也是一次 HREADY 握手的读)
ID: 译码,识别 STR
EX: 算好地址 0x40011004,数据 r1 就绪

core 驱动总线:
  HADDR  = 0x40011004
  HWDATA = r1
  HWRITE = 1
  HSIZE  = 2(字)

▼ 总线矩阵译码:0x40010000~0x40013FFF → 属于 APB2 上的设备
▼ AHB→APB2 桥(协议翻译,见下)
▼ USART1 收到写 → 字节进 DR → 开始发送
```

地址相和数据相各一拍,正常完成:

```text
clk    _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
HADDR  ────<0x40011004>───────
HWDATA ──────<data>───────────
HWRITE ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
HREADY ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
        ↑地址相       ↑数据相:完成,str 退休
```

slave 忙(如 FMC 写 FIFO 满)→ 拉低 HREADY,插入等待周期:

```text
clk    _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
HADDR  ────<0x40011004>────────────────  ← 所有信号保持
HREADY ‾‾‾‾‾‾‾‾‾‾\______________‾‾‾
                     ↑ 等待周期:整条流水线冻结
```

MCU 级核没有乱序、没有别的指令可跑,**PC 不走、寄存器不回写,大家一起等这根线**。
所以 MCU 上访问慢外设的代价是精确可见的周期数。

**AHB→APB 桥的反压传导**:外设几乎都挂在 APB 上,桥把 AHB 的"地址相+数据相"翻译成
APB 的"setup 相+access 相",并把 APB 的 PREADY 反向映射成 HREADY:

```text
APB2 侧(UART 看到的):
PSEL    ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
PENABLE ________‾‾‾‾‾‾‾‾‾‾‾
PADDR   ────<0x40011004>───
PWDATA  ────<data>─────────
PREADY  ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾  ← UART 收下了才拉高
```

```text
反压链路:UART 没准备好 → PREADY 低 → 桥不能回 HREADY 高 → core 流水线冻结
```

一句话概括 MCU 级交互:**core 是 master 主动驱动信号;slave 用一根 HREADY 叫停;
叫停期间整条流水线陪等**。store buffer、posted write、异步完成,全是高性能核为了
不"陪等"而加出来的机制。

### 配置②:RV32,有 cache,无乱序,无 store buffer

没有 store buffer → store 必须真正"做完"才退休;有 cache → **"做完"的终点从总线变成了 cache**。
命中时总线根本不知道有这次访问。

```text
sw x5, 0(x6)
   │
   ▼
┌──────────┐     ┌─────────────────────┐      ┌──────────┐
│  LSU     │────▶│  D-cache 控制器      │─────▶│  总线     │
│  addr+data│    │  (查 tag,管 SRAM,    │ miss │  互连    │
└──────────┘     │   自己是个总线 master) │      └──────────┘
                 └─────────────────────┘
```

新角色:**cache 控制器是总线上自主的 master**。以前总线上只有"当前这条指令"产生的交易;
现在多了两类没人执行指令时也发生的交易:miss 时的 fill(读整行回来)和 write-back 时的
dirty 行写回——此时流水线可能在跑毫不相干的指令。

sw 的三条路(以 write-back + write-allocate 为例):

```text
sw x5, 0(x6)
   │
   ▼
查 tag ──┬─ 命中 ──▶ 写 cache SRAM,标 dirty ──▶ 1 拍,退休
         │                                        └─ 总线零交互!
         │
         ├─ miss ──▶ 总线读事务 fill 整行(32/64B)──▶ 再写进 line
         │                 │                           标 dirty → 退休
         │                 └─ 流水线停整个 fill 延迟(无 buffer 可扛)
         │
         └─ MMIO 地址(PMA 非缓存)──▶ 直通总线,等 HREADY 完成才退休
                                       └─ 和 MCU 级完全一样
```

| 路径 | store 等多久 | 总线参与吗 |
|------|-------------|-----------|
| cache 命中(write-back) | 1 拍 | 完全不 |
| cache miss | 整个 line fill | 是(一次**读**!) |
| MMIO | 总线握手完成 | 是(直通) |

- miss 那条路的诡异之处:**你发的是写,总线上发生的是读**(write-allocate 先把整行读回来再改)。
  no-write-allocate 则 miss 时直接写穿到总线(≈ MCU 行为),由设计权衡决定。
- MMIO 因为 PMA 非缓存,退回 MCU 级的同步阻塞——**同一个核上两种 store 语义并存,靠地址区分**。
- "完成"三层次(见 [[write-handshake]])重新对齐:命中时 ① = 写进 cache SRAM 即退休,
  而 ③(数据真正进 DRAM)**推迟到未来某次 eviction**——dirty 行被替换时由 cache 控制器
  自主写回。posted write 的"债"从 store buffer 转移到了 **dirty cache line**。
- **DMA 场景因此变了**:store buffer 不存在,但要担心 dirty 行没写回。DMA 来读 buffer 前,
  用 cache 管理指令(Zicbom 的 `cbo.clean`/`cbo.flush`,或 cache 控制寄存器)刷 dirty 行。
- `fence` 在这配置下几乎无事可做:没有 store buffer 可排,没有乱序可约束。
  这也是 fence 和 cache flush 是两码事的来源:fence 排空的是微架构缓冲,不管 cache。
- **若配 write-through:直接退化回 MCU 行为**。每个 store 都要等总线握手完成才退休,
  有 cache 也没救——写密集循环的周期数 = 总线写延迟 × 次数。无 store buffer 的核
  要么忍受这个,要么必然选 write-back。

### 配置③:高性能核(有 cache,有 store buffer)

完整链路:**流水线打包 → store buffer 缓冲 → BIU 握手 → 译码路由 → slave 应答**。

一条 sw 穿过流水线:

```text
┌───────────────┐
│ IF            │  I-cache 取出: sw x5, 0(x6)
└──────┬────────┘
       ▼
┌───────────────┐
│ ID            │  译码:S-type;读寄存器 rs1=x6(基址), rs2=x5(数据)
└──────┬────────┘
       ▼
┌───────────────┐
│ EX            │  ALU: addr = x6 + signext(imm) → 有效地址
└──────┬────────┘
       ▼
┌───────────────┐
│ MEM           │  LSU 打包 {addr, data, size} 申请写 store buffer
└──────┬────────┘   ├─ buffer 满 → 流水线停在这里(stall,这就是"反压")
       ▼            └─ 有空位 → 写入,指令 retire → core 视角的"完成"
┌───────────────┐
│ WB            │  sw 不写回 rd,空操作
└───────────────┘
```

`size` 来自指令编码:`sb/sh/sw` 决定写 1/2/4 字节,之后翻译成总线的字节选通。

core ↔ 总线交界:

```text
┌─────────────────────── core ───────────────────────┐
│                                                     │
│   store buffer(按序 FIFO)                            │
│   ┌────────────┬────────────┬────────────┐          │
│   │ entry 0    │ entry 1    │ entry 2    │  ...     │
│   │ addr,data, │ addr,data, │ addr,data, │          │
│   │ size,valid │ size,valid │ size,valid │          │
│   └─────┬──────┴────────────┴────────────┘          │
│         │ BIU 取最老 entry,按序发起总线事务            │
│         ▼                                            │
│   ┌───────────────────────────┐                      │
│   │ Bus Interface Unit (BIU)  │                      │
│   │ 维护 VALID/READY 握手       │                      │
│   └─────────────┬─────────────┘                      │
└─────────────────┼────────────────────────────────────┘
                  ▼
      ┌────────────────────────┐
      │ 总线互连(地址译码路由)   │
      └───────────┬────────────┘
                  ▼
        slave(FMC / UART / DRAM...)
```

职责分工:**流水线只负责把 store 塞进 buffer**;buffer 之后的所有总线事务由 BIU 异步完成。
两者唯一耦合点是"buffer 满 → stall"。

总线上的握手时序(AXI 一次写 = 三通道依次握手,规则都是 **VALID/READY 同时为高才采样**):

```text
clk    _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
AWADDR ────<0xC0001234>──────────────────────
AWVALID ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
AWREADY ________‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
               ↑ 地址被采样(地址相)

WDATA  ─────────────<data>──────────────────
WVALID ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
WREADY ________‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
               ↑ 数据被采样(数据相)

BVALID ______________‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
BRESP  ───────────────<OKAY>───────────────
BREADY ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
               ↑ 响应被 core 采样 → 该 entry 释放
```

全过程时序关系:

```text
时间 →
  ① sw 进 MEM 阶段,写 store buffer,retire     ← core 视角完成(可能只需 1 拍)
  ② BIU 发起 AW 握手(发地址)
  ③ BIU 发起 W 握手(发数据)
  ④ 等待 B 响应(slave 收下了)
  ⑤ entry 释放,buffer 前进一格
```

① 和 ②③④⑤ 异步并行:②还没开始时,流水线已经在跑后面的指令了。

关键设计点:

- **store-to-load forwarding**:后续 load 读同一地址时,数据直接从 store buffer 拿,不用等它
  写到总线上——这就是"store 不用等完成,自己却读得到新值"的原因。
- **fence/DMB 的微架构含义**:强制"等 buffer 排空"(等 ⑤ 发生)才让后续指令继续。
- **load 的交互完全不同**:load 在 MEM 阶段要同步等数据(BIU 发起读事务 → 数据回来 →
  才写回寄存器),所以 load 是"阻塞"的,store 是"posted"的。
- **字节选通**:`sb` 变成 AXI 的 `WSTRB=0b0001`(按地址选位)或 AHB 的 `HSIZE=0`,
  从指令 size 字段翻译而来。

### 三种配置总对比

| | 无 cache 无 buffer(MCU 级) | 有 cache 无 buffer | 有 cache 有 buffer(高性能) |
|---|---|---|---|
| store 命中退休 | 等总线握手 | 1 拍(写 cache) | 进 buffer 即退休 |
| store 拖住流水线? | 每次都拖 | 仅 miss 和 MMIO 拖 | 几乎不拖 |
| 总线上的写来自 | 每次 store | 仅 eviction/穿写 | 异步排空 |
| ③落地的"债" | 无(当场落地) | dirty cache line | dirty line + buffer |
| DMA 前要做什么 | 排空(若有 buffer) | clean cache | 排空 + clean |

## 要点

- MCU 级:信号组 + 一根 HREADY,一拍一拍握手;slave 拉低 HREADY = 整条流水线冻结陪等;
  反压经 APB 桥的 PREADY→HREADY 逐级传导;周期代价精确可见
- 有 cache 无 buffer:"做完"的终点从总线变成 cache;命中零总线交互,miss 触发 fill(读),
  MMIO 退回 MCU 行为;债从 store buffer 变成 dirty line;write-through 会退化回 MCU 级
- 高性能核:流水线打包 → store buffer 缓冲 → BIU 异步握手 → 译码路由 → slave 应答;
  流水线与总线事务解耦,唯一耦合是 buffer 满反压
- 三种配置的共同点:靠地址(译码/PMA)决定这次 store 走哪条路

## 参考

- [mmio.md](./mmio.md) — MMIO 机制
- [write-handshake.md](./write-handshake.md) — "完成"的三个层次
- [fmc-sdram.md](./fmc-sdram.md) — FMC SDRAM 具体实例
- AMBA AXI / AHB / APB 规范
