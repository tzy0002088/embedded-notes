# embedded-notes

嵌入式学习笔记：QEMU 建模、RISC-V、xv6 内核、SoC/MCU 总线、Linux 内核。学习过程中产生的提问、理解和笔记，按方向分目录沉淀。

## 目录

| 方向 | 目录 | 记录数 | 说明 |
|------|------|--------|------|
| QEMU | [qemu/](./qemu/_index.md) | 4 | QEMU 整体架构与核心子系统 |
| ⤷ QEMU 内存模型 | [qemu-memory/](./qemu-memory/_index.md) | 0 | MemoryRegion、AddressSpace、IOMMU |
| RISC-V | [riscv/](./riscv/_index.md) | 1 | RISC-V 架构与实现 |
| xv6-riscv | [xv6-riscv/](./xv6-riscv/_index.md) | 3 | xv6 操作系统内核分析 |
| SoC/MCU 总线 | [soc-bus/](./soc-bus/_index.md) | 8 | 总线互连、MMIO、写事务、store 数据通路、协议桥、DMA、USB DMA 实例 |
| Linux 内核 | [linux-kernel/](./linux-kernel/_index.md) | 1 | 中断、softirq、threaded IRQ、USB gadget 驱动实例 |

> 新增方向时：新建子目录 → 创建 `_index.md` → 在上表加一行。

## 记录模板

新建记录时复制以下内容到 `.md` 文件：

```markdown
---
title: ""
date: YYYY-MM-DD
tags: []
status: draft   # draft | notes | polished
---

## 背景

（为什么会有这个问题？在做什么的时候遇到的？）

## 提问

（把当时的困惑、问题列出来）

## 理解

（学到的答案、自己的理解）

## 要点

- 
- 

## 参考

- 
```
