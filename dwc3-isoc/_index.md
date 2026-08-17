# DWC3 等时传输（Isochronous）

DWC3 USB 控制器的等时传输：硬件机制、事件/中断体系、驱动层策略与异常处理。
背景：Zephyr 上为 UAC2 音频流实现 `udc_dwc3.c` 等时支持。

## 记录列表

| 日期 | 记录 | 状态 | 说明 |
|------|------|------|------|
| 2026-08-16 | [hardware-model.md](./hardware-model.md) | notes | 硬件机制：间隔/BD/TRB 规则、起始时间同步、ZLP 行为矩阵、错过语义、两种端点模型 |
| 2026-08-16 | [events-interrupts.md](./events-interrupts.md) | notes | 事件与中断体系：DEPEVT/DEVT、使能位、Bus Expiry、事件环 |
| 2026-08-16 | [driver-strategy.md](./driver-strategy.md) | notes | Linux dwc3 驱动策略 + 驱动层操作手册（含异常速查表） |
