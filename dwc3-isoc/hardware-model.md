---
title: "DWC3 等时传输硬件机制：间隔、BD、起始同步与错过语义"
date: 2026-08-16
tags: [usb, dwc3, isochronous, zephyr, uac2]
status: notes
---

## 背景

在 Zephyr 上给 UAC2 音频流做 DWC3 等时传输支持，精读 DWC_usb3 Programming Guide
3.30b 第 4.3 章（中文翻译：[DWC_usb3_programming_4.3_isoc_zh.pdf](../../work/ls_zephyr/DWC_usb3_programming_4.3_isoc_zh.pdf)，
原文第 355–363 页）。本控制器只支持 memory-based 模型（无外部 FIFO 直连）。

## 提问

1. 主机是在固定的 interval 内发 IN/OUT token 吗？设备没在间隔内回数据怎么办？
2. 传输怎么"对齐"到主机的服务间隔？起始时间怎么定？
3. 错过了间隔，硬件给软件什么交代？
4. 等时端点有哪些"型号"？

## 理解

### 协议层：无握手、无重传、ZLP 兜底

- 主机在每个服务间隔的固定微帧发 IN token（或 OUT 数据包），带宽是保留的，
  但**没有 ACK/NAK 握手机制**：设备必须当场回包，**没数据也要回零长度包（ZLP）**。
- 主机无从区分"没数据"与"数据丢失"——看到的只是数据流里的空洞；
  数据**从不重传**，空洞由上层（音频 underflow 处理等）消化。
- 所以"保证服务"只保证**轮询机会**，不保证数据送达。

### 核心概念

| 概念 | 含义 |
|---|---|
| 服务间隔 | `2^(bInterval−1)` 微帧（HS/SS 下 1 µF = 125 µs），主机每个间隔轮询一次 |
| 缓冲描述符 BD | **一个间隔**的传输单元 = 一串 TRB（首 TRB `ISOCHRONOUS_FIRST`，其余 CHN=1 链接） |
| TRB 环 | 循环链表；**至少 2 个 BD**（4.3.8 注意：硬件卡在过期 TRB 上会一直发 ZLP） |
| 预取窗口 | memory-based IN：硬件**最早在间隔开始前一个间隔**取 TX 数据——TRB 必须提前 2 个间隔就绪 |
| Start Transfer 时间 | 启动命令参数里指定的未来帧号，硬件从该间隔起开始消耗 BD |

### 起始时间同步（等时传输的生命线）

- **传输未激活**时主机 poll：硬件回 ZLP + 产生 **XferNotReady**，事件**高 16 位 = 微帧号**；
  每轮传输**只产生一次**，发过 END_TRANSFER 后才重新武装。
- 软件拿微帧号算出未来的对齐间隔，发 **START TRANSFER（参数 = 帧号）**；
  命令一发即激活，起始点之前主机的 poll 由硬件**静默回 ZLP**（无事件、不碰 TRB）。
- 起始时间窗口约 **[当前, 当前+4s]**：过期或过远都报 Bus Expiry，传输不启动。

### 间隔内 ZLP 行为矩阵（4.3.6 IN）

| 情况 | 总线行为 | 事件 |
|---|---|---|
| 数据已备好 | 发数据；TRB BUFSIZ 逐包递减，减到 0 回收（IOC 置位则报 XferInProgress） | XferInProgress(IOC) |
| 起始时间之前被 poll | 静默 ZLP | 无 |
| 期望间隔内 poll 但数据没备好 | ZLP（传输已激活，无 XferNotReady） | **下一间隔开始**报 XferInProgress(MissedIsoc=1) |
| 间隔内数据已发完又被 poll 一次 | ZLP | 无 |
| 间隔没发完（主机轮询不足/数据超量） | Flush TxFIFO | A、B 两 BD 以 MissedIsoc 回收 |

### 错过语义（Missed Isochronous）

- 错过**在下一个间隔开始时**才报告：`XferInProgress` 事件 MissedIsoc 位（BIT 15）；
- **间隔 A 和 B 两个 BD 一起被回收**（丢一个因主机/数据，第二个是设备重新同步到
  间隔 C 的代价）——"一次丢两个间隔是正常现象"，驱动不要试图救回 B；
- 回写细节（4.3.7）：首 TRB 回收；**中间 CHN=1 非首 TRB 不写回、HWO 保持 1**（软件可直接收回复用）；
  尾 TRB HWO=0、BUFSIZ=剩余量、TRBSTS=MissedIsoc；
- **MissedIsoc 置位时 BUFSIZ 不可信**——不能当实际收发字节数上报；
- 是否产生中断由尾 TRB 的 IOC/IMI 决定（见 [events-interrupts.md](./events-interrupts.md) 表 4-11）；
  两个位都没设 → 错过**完全静默**，只能轮询环上 HWO 位发现。

### 两种端点模型（表 4-12）

| | memory-based（本控制器） | FIFO-based（DEPCFG Param1[31]=1） |
|---|---|---|
| TRB 指向 | 系统内存缓冲区 | 地址译码信号：读地址 X ⇒ 对外部 FIFO X 发 pop |
| 数据路径 | 控制器 DMA：内存 ⇄ 内部 TxFIFO | 外部硬件 FIFO（codec/ISP）经地址翻译逻辑直连，CPU/RAM 不参与 |
| TRB 写回 | 有（HWO/BUFSIZ/TRBSTS） | 永不写回；HWO 恒视为 1 |
| 数据有效要求 | TRB 提前 2 间隔排好 | 数据在目标间隔开始前 ≥1 µF 进 FIFO（乒乓双组） |
| 错过指示 | 事件 + TRB 写回 | 仅 XferInProgress（IOC/IMI）；无实际传输量指示；跳 1-2 个 pop，外部逻辑刷新重填 |

## 要点

- 等时可靠性 = 三层配合：协议层 ZLP 兜底、控制器层 XferNotReady/MissedIsoc 语义、软件层提前排队。
- 起始时间**只能**从 XferNotReady 的微帧号推算；裸启动必撞 Bus Expiry。
- 错过不丢同步（硬件自动跳到间隔 C），只丢数据；FIFO 型连相位都要外部逻辑兜底。
- memory-based 是 Linux 全系唯一支持的模型；FIFO-based 连 Linux 驱动都没实现（只有空事件 case）。

## 参考

- DWC_usb3 Programming Guide 3.30b 4.3 章（原文 355–363 页，中文翻译 PDF 同上）
- Linux dwc3 驱动：`/home/tzy/inspiration/qemu-linux/linux/drivers/usb/dwc3/gadget.c`
- USB 2.0 Spec §5.6.4 / §5.9.2（等时事务、DATA PID 顺序）
