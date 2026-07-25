---
title: "xv6-riscv 学习笔记"
date: 2026-07-19
tags: [xv6, riscv, os, kernel]
---

## 文档列表

| 文档 | 说明 |
|------|------|
| [编译构建系统](build-system.md) | Makefile 架构、交叉编译工具链、编译标志、链接脚本、QEMU/GDB |
| [启动流程](boot-process.md) | PMP、Sv39 MMU、页表、内核→用户态切换、文件系统加载、Shell 启动 |
| [内核页表初始化](kernel-vm.md) | kvmmake 各映射详解、直接映射设计、内核页表 vs 用户页表 |
| [Sv39 三级页表深入](sv39-paging.md) | VA/PA 位布局、PX 宏、walk 动态分配、PTE 格式、大页 |
| [用户态程序](user-program.md) | ELF 格式、静态链接、kexec 加载流程、与 Linux 的对比 |
| [usys.pl 系统调用桩生成](usys-pl.md) | 脚本用途、汇编桩原理、系统调用完整链路 |
| [Virtio 磁盘驱动与中断](virtio-disk.md) | Virtio 协议、Virtqueue、PLIC、完整中断路径 |
| [内核启动与底层细节问答](startup-and-lowlevel.md) | 链接脚本、medany、push_off 设计、启动委托、多核 fence 等 12 个问答 |
| [Sv39 页表深入、内存映射与缺页处理](sv39-page-table-internals.md) | PTE 字段、叶子/大页、PLIC 64MB 映射过程、内核栈与 guard page、缺页处理 |

## 源码位置

`/home/tzy/inspiration/xv6-riscv/`

## 关键文件速查

| 文件 | 作用 |
|------|------|
| `kernel/entry.S` | 内核入口（汇编），第一行代码 |
| `kernel/start.c` | M-mode 启动，配置 PMP，mret 到 S-mode |
| `kernel/main.c` | 内核主函数，初始化各子系统 |
| `kernel/vm.c` | 虚拟内存管理，Sv39 三级页表 |
| `kernel/trampoline.S` | 用户↔内核切换的蹦床代码 |
| `kernel/trap.c` | 陷阱/中断/系统调用分发 |
| `kernel/proc.c` | 进程管理、调度器、forkret |
| `kernel/exec.c` | ELF 加载（kexec） |
| `kernel/virtio_disk.c` | Virtio 块设备驱动 |
| `kernel/plic.c` | PLIC 中断控制器 |
| `kernel/kernel.ld` | 内核链接脚本 |
| `kernel/riscv.h` | RISC-V 寄存器操作和宏 |
| `kernel/memlayout.h` | 物理/虚拟内存布局常量 |
| `user/init.c` | init 进程（fork + exec sh） |
| `user/ulib.c` | 用户库（start 入口、libc 函数） |
| `user/usys.pl` | 生成系统调用汇编桩 |
| `Makefile` | 唯一构建文件 |
