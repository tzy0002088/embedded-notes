# SoC/MCU 总线级基础

SoC/MCU 内部总线互连、MMIO、外设交互、总线协议等硬件底层知识学习记录。

## 记录列表

| 日期 | 记录 | 状态 | 说明 |
|------|------|------|------|
| 2026-08-15 | [mmio.md](./mmio.md) | notes | MMIO 机制:core 如何通过寄存器与外设打交道 |
| 2026-08-15 | [write-handshake.md](./write-handshake.md) | notes | 总线写事务的完成信号与"完成"的三个层次 |
| 2026-08-15 | [sw-to-bus.md](./sw-to-bus.md) | notes | 从 sw 指令到总线交易:core 与总线交互全链路(三种芯片配置) |
| 2026-08-15 | [fmc-sdram.md](./fmc-sdram.md) | notes | FMC SDRAM 具体实例:两级缓冲、波形翻译、两种语义 |
| 2026-08-15 | [bus-protocols.md](./bus-protocols.md) | notes | 协议桥:core 内部方言如何翻译成 AXI/AHB/APB 事务 |
| 2026-08-15 | [bus-masters.md](./bus-masters.md) | notes | DMA master 与多 master 世界:双端口 IP、仲裁、一致性根源 |
| 2026-08-15 | [glossary.md](./glossary.md) | notes | 术语表:core/AMBA/RISC-V/Cortex-M/存储外设 |
