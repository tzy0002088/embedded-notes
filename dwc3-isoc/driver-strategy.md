---
title: "DWC3 等时传输驱动策略：Linux 做法与操作手册"
date: 2026-08-16
tags: [usb, dwc3, isochronous, linux, zephyr, driver]
status: notes
---

## 背景

参考 Linux dwc3 gadget 驱动（`/home/tzy/inspiration/qemu-linux/linux/drivers/usb/dwc3/gadget.c`）
总结等时传输的驱动策略，形成 udc_dwc3.c 的操作手册。前置知识：
[hardware-model.md](./hardware-model.md)、[events-interrupts.md](./events-interrupts.md)。

## 提问

Linux 驱动怎么把 XferNotReady/MissedIsoc/Bus Expiry 这些机制串成一个可靠的状态机？
落在自己的驱动里，每一步该做什么、每种异常怎么处理？

## 理解

### Linux 的核心策略（附代码位置）

1. **绝不预启动**（gadget.c:2024-2037）：isoc 端点 enable/queue 后不发 START TRANSFER，
   必须等 XferNotReady 拿微帧号——否则极大概率 Bus Expiry。
   请求先挂 pending_list；若 XferNotReady 已来过（PENDING_REQUEST 标志）则立即启动。
2. **启动 = 读帧号 + 对齐 + 带时启动**（1914-1978）：
   - XferNotReady 参数 → `dep->frame_number`；bInterval≤14 时重读 DSTS.SOFFN 低 14 位
     与高 2 位合并防过期（1934-1953）；
   - `DWC3_ALIGN_FRAME`：`(frame + n×interval) & ~(interval-1)` 对齐到间隔边界（30-31）；
   - n 从 1 起逐次 +1 重试（最多 5 次），bInterval<3 额外加量（≥500 µs 调度余量）；
   - **START TRANSFER 命令参数 = 帧号**（1710）；返回 -EAGAIN（Bus Expiry）就换下一个间隔。
3. **运行期补环**：已启动的端点有新 TRB 入环 → **UPDATE TRANSFER + resource index**（1711-1713）。
4. **TRB 构造**（1311-1421）：首 TRB `ISOCHRONOUS_FIRST`、尾 TRB **IOC+IMI**（1359、1392）、
   HS 按长度算 PCM1 多事务（1342-1354）、**wmb() 后置 HWO**（1416，数据手册 4.2.3.2）。
5. **错过即整体重启**（3714-3717）：MissedIsoc（-EXDEV，3767-3768）→
   **End Transfer + ForceRM**（不回写 TRB 进度、不清 HWO，驱动自己清 HWO 回收）→
   EPCMDCMPLT 清理 → 等下一个 XferNotReady 重新走启动流程。
   即：**不用硬件"跳间隔 C 自同步"行为，选择整体推倒重来换精确重同步。**
6. **no_interrupt 请求**：尾 TRB 不设 IOC/IMI 时，回收靠读 TRBSTS 回读（MISSED_ISOC→-EXDEV，3621-3638）。
7. **不支持 FIFO-based**：全驱动无 FIFO_BASED 使用；RXTXFIFOEVT 事件空 case 忽略
   （gadget.c:3929、ep0.c:1218）——本控制器同样只做 memory-based。

### 驱动层操作手册

**① 端点使能（一次配置）**：DEPCFG（isoc 类型、MaxPacket、FIFO Number、BINTERVAL_M1、
Param1[31] FIFO-based=0、开 XferNotReadyEn/XferInProgressEn/XferCompleteEn）；
DEVTEN（USBRST/DISCINT/CONNECT_DONE，建议 ULStChg/EVNTOVERFLOW，**SOF 不开**）；
TRB 环 ≥2 BD（实际 4-8 个 request 深度）；首启不发 START TRANSFER。

**② 启动**：XferNotReady → 读微帧号 → 刷新 → 对齐（n≥2 个间隔）→ START TRANSFER(帧号)。
Bus Expiry → n+1 重试（≤5 次）→ 全败则 End Transfer 等下一个 XferNotReady。

**③ 运行**：XferInProgress(IOC) 回收 request → giveback → 上层补 request → UPDATE TRANSFER。
队列深度就是节拍，无需任何周期中断。

**④ 异常速查表**：

| 场景 | 硬件行为 | 驱动动作 |
|---|---|---|
| 起始时间前被 poll | 静默 ZLP，无事件 | 无需处理 |
| 间隔内数据没备好 | ZLP；下间隔开始报 XferInProgress(MissedIsoc)；A、B 双回收；BUFSIZ 不可信 | 上抛 -EXDEV；丢弃数据；重排间隔 C；或 End Transfer + 重启 |
| 间隔没发完 | Flush TxFIFO + A、B 双回收跳 C | 同上 |
| 同间隔多 poll 一次 | ZLP，无事件 | 无需处理 |
| OUT 主机静默 | **无任何事件**（未来间隔来包才察觉） | 消费者 underflow / 定时器 → End Transfer → 等 XferNotReady |
| OUT 收 CRC 错包 | 期望序列号不递增，同间隔后续包全丢 | 按错过处理 |
| OUT 缓冲不足（HWO=0） | 无 XferNotReady；等 Update Transfer；A 按 MissedIsoc 回收 | 补 TRB + Update Transfer |
| Bus Expiry | 命令被拒、传输未启动 | 推后重试 / 整体重启 |
| USBRST / DISCONNECT | 帧号体系重置 | 全端点 End Transfer、清环、重配 |
| ULStChg（U1/U2 退出） | 低功耗返回，总线时间可能跳变 | isoc 端点保守重启重对齐 |
| 事件环溢出 | 事件丢失 | 清环 + 轮询 TRB 状态重建认知 |

**⑤ 停止**：isoc 无需 LST=1 优雅结束；End Transfer → Command Complete；
强制停用 ForceRM + 软件清 HWO 回收；停后要么等 XferNotReady 重启，要么 ep_disable。

### Zephyr UDC 映射要点

- `ep_enqueue`：备 TRB（首/尾规则）+ 入 pending；kick 按状态机：未启动→等 XferNotReady
  （或 PENDING_REQUEST 已置则立即启动）；已启动→UPDATE TRANSFER。
- IRQ 分发：DEPEVT 按事件类型进 XferNotReady / XferInProgress / XferComplete /
  EPCMDCMPLT 四个 handler；DEVT 管复位/连接。
- 两条铁律：① 尾 TRB 必设 IOC/IMI，错过不能静默；② 起始时间只能来自 XferNotReady
  的微帧号，任何裸启动都会撞 Bus Expiry。
- MissedIsoc 恢复初期建议直接抄 Linux 的"End Transfer + 重启"方案：状态简单、重同步干净；
  熟练后再考虑硬件自同步（跳 C）省一次状态机往返。

## 要点

- 时间对齐全在**软件侧**（帧号推算 + 对齐 + 重试），不信任硬件自动对齐——Bus Expiry 的代价比多算一步贵。
- 错过处理本质是选择：硬件自同步（丢两个间隔、状态简单）vs 整体重启（多一套 END_TRANSFER
  状态机、重同步精确）。Linux 选后者。
- 可靠性的底线组合：XferNotReady（起）+ IOC/IMI（每间隔可见）+ -EXDEV 上抛（上层可知）+ EVNTOVERFLOW（兜底）。

## 参考

- `qemu-linux/linux/drivers/usb/dwc3/gadget.c`（行号见上文）
- 翻译 PDF 4.3.5-4.3.11（启动/间隔行为/检查状态/添加间隔/事件节制/结束传输）
- 本仓库 [hardware-model.md](./hardware-model.md)、[events-interrupts.md](./events-interrupts.md)
