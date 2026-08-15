# 当前学习状态

> 新会话开始时，先读这个文件就能接上。

## 正在进行

- **系统学习 QEMU**，目标：能建模自研 RISC-V SoC + Board
- 已读完理论部分（QOM、MemoryRegion、SysBus、spike.c 逐行分析）
- **下一步：动手实践**，从零构建一个最小 RISC-V 机器，逐步加功能
- 环境就绪：QEMU build/ 已有 ninja，交叉编译器 riscv64-unknown-elf-gcc 可用
- **同时开始学习 SoC/MCU 内部总线级基础知识**：MMIO 机制、总线互连、写事务握手已沉淀到 [soc-bus/](./soc-bus/_index.md)

## 待解答的问题

- （学习中产生的问题记录在这里，解答后移到对应方向的 .md 记录里）

## 最近更新

| 日期 | 记录 | 方向 |
|------|------|------|
| 2026-07-02 | deep-dive-foundations.md | qemu |
| 2026-07-02 | learning-roadmap.md | qemu |
| 2026-07-02 | aspeed-bmc.md | qemu |
| 2026-07-08 | architecture-overview.md | qemu |
| 2026-07-08 | build-compile.md | qemu |
| 2026-07-02 | sensor-emulation.md | qemu |
| 2026-07-02 | boot-flow.md | riscv |
| 2026-08-15 | mmio, write-handshake, sw-to-bus, fmc-sdram, bus-protocols, bus-masters, glossary | soc-bus |
| 2026-08-16 | usb-dma-path | soc-bus |
