---
title: "SoC/MCU 总线术语表"
date: 2026-08-15
tags: [soc, bus, glossary]
status: notes
---

持续补充中的术语表,按类别分组。

## core 内部 / 微架构

| 缩写 | 全称 | 意思 |
|------|------|------|
| LSU | Load/Store Unit | 装载/存储单元,core 里执行 load/store 的功能部件,打包地址和数据发往缓存/总线 |
| BIU | Bus Interface Unit | 总线接口单元,core 与总线之间的桥,维护握手、跟踪未完成事务 |
| IF/ID/EX/MEM/WB | Instruction Fetch/Decode/Execute/Memory/WriteBack | 经典五级流水线:取指/译码/执行/访存/写回 |
| retire | (指令)退休 | 指令执行完成、正式离开流水线,"架构上做完了" |
| stall | 停顿 | 流水线因等待(总线忙、依赖未满足)而停住 |
| store buffer | 存储缓冲 | core 内暂存待发 store 的小队列,让 store 不等总线 |
| OoO / in-order | Out-of-Order / 顺序执行 | 乱序执行 / 按程序顺序执行 |
| I-cache / D-cache | Instruction/Data cache | 指令缓存 / 数据缓存 |
| write-back / write-through | 回写 / 写穿 | 写命中 cache 时:只改 cache 标 dirty / 同时写穿到总线 |
| write-allocate | 写分配 | store miss 时先把整行读进 cache 再写 |
| eviction / dirty line | 淘汰 / 脏行 | cache 行被替换出去 / 被改过还没写回内存的行 |

## 总线协议(AMBA 家族)

| 缩写 | 全称 | 意思 |
|------|------|------|
| AMBA | Advanced Microcontroller Bus Architecture | ARM 定义的总线协议族(AXI/AHB/APB 都属它) |
| AXI | Advanced eXtensible Interface | 高性能总线,5 通道、支持 burst/乱序/outstanding,SoC 主干 |
| AHB | Advanced High-performance Bus | 中速总线,单地址/数据通道,靠 HREADY 握手 |
| APB | Advanced Peripheral Bus | 慢速外设总线,两相时序、低功耗低面积 |
| AHB-Lite | — | AHB 为单 master 场景的精简版(Cortex-M 直出这个) |
| master / slave | 主设备 / 从设备 | 发起事务的一方 / 响应事务的一方 |
| VALID/READY | — | 握手协议:双方同时为高才完成一拍,谁没准备好就等谁 |
| burst | 突发传输 | 一次事务连续传多个数据拍 |
| outstanding | 在途未完成 | 允许同时挂着多笔未完成事务(不等上一笔响应就发下一笔) |
| posted write | 先行写 | 发出后不等响应就继续的写(先斩后奏) |
| backpressure | 反压 | slave 忙时用握手信号把压力逐级传回,最终停住流水线 |

信号名规律:H 开头 = AHB(`HADDR`、`HWDATA`、`HREADY`),P 开头 = APB(`PSEL`、`PENABLE`、`PREADY`),
AXI 的五个通道:写 = `AW`(地址)/`W`(数据)/`B`(响应),读 = `AR`(地址)/`R`(数据)。

## RISC-V 相关

| 缩写 | 全称 | 意思 |
|------|------|------|
| S-type | Store 指令格式 | RISC-V 的 store 指令编码格式(sb/sh/sw) |
| PMA | Physical Memory Attributes | 物理内存属性,规范里标记某地址区域是内存还是 MMIO、可否缓存/推测 |
| fence | — | RISC-V 内存屏障指令,排空缓冲、约束访存顺序 |
| WSTRB | Write strobe | 写字节选通,标记这次写覆盖哪些字节 |
| Zicbom | — | RISC-V 缓存管理扩展名,提供 cbo.clean/cbo.flush 等指令 |
| CMO | Cache Management Operation | 缓存管理操作(clean/invalidate 一类) |
| TileLink | — | SiFive 的片上互连协议,带缓存一致性语义,Rocket 的"母语" |
| PLIC | Platform-Level Interrupt Controller | 平台级中断控制器 |

## ARM / Cortex-M 相关

| 缩写 | 全称 | 意思 |
|------|------|------|
| ICode / DCode / S-bus | — | Cortex-M 的三条总线接口:取指 / 数据 / 系统(外设走 S-bus) |
| MPU | Memory Protection Unit | 内存保护单元,Cortex-M 上配置区域属性(可否缓存/访问权限) |
| DMB / DSB | Data Memory Barrier / Data Synchronization Barrier | ARM 的内存屏障指令:约束顺序 / 排空写缓冲 |
| ART | Adaptive Real-Time accelerator | ST 的 Flash 加速器(预取 + 小缓冲),只对内部 Flash 生效 |

## 存储与外设

| 缩写 | 全称 | 意思 |
|------|------|------|
| SDRAM | Synchronous DRAM | 同步动态随机存取存储器,需要控制器定期刷新 |
| FMC | Flexible Memory Controller | STM32 的外部存储器控制器,SRAM/SDRAM/NOR/NAND 都归它管 |
| CS#/RAS#/CAS#/WE# | Chip Select / Row/Column Address Strobe / Write Enable | SDRAM 四根控制线,组合出 ACT/READ/WRITE/PRE/REF 命令 |
| CL(CAS latency) | Column Address Strobe latency | 发出读命令到数据出现在 DQ 上的周期数 |
| tRCD | RAS to CAS Delay | 行激活到列访问的最小间隔 |
| DQ | Data I/O pins | SDRAM 的数据引脚 |
| SDCR/SDTR | SDRAM Control/Timing Register | FMC 里配 SDRAM 参数(行/列位数、时序)的寄存器 |
| THR / LSR | Transmit Holding / Line Status Register | UART 的发送保持寄存器 / 线状态寄存器 |
| MMIO | Memory-Mapped I/O | 内存映射 I/O,外设寄存器映射进地址空间 |
| DMA | Direct Memory Access | 直接内存访问,不经 CPU 在设备间搬数据 |
| DTB | Device Tree Blob | 设备树二进制,启动时告知内核硬件布局 |

## 相关记录

- [mmio.md](./mmio.md) — MMIO 机制
- [write-handshake.md](./write-handshake.md) — 完成三层次
- [sw-to-bus.md](./sw-to-bus.md) — store 数据通路
- [fmc-sdram.md](./fmc-sdram.md) — FMC SDRAM 实例
- [bus-protocols.md](./bus-protocols.md) — 协议桥与三种总线分工
- [bus-masters.md](./bus-masters.md) — DMA master 与多 master 世界
