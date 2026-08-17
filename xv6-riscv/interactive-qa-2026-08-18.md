---
title: "xv6 交互问答：系统调用桩、UART 中断、usertrap 与 copyin"
date: 2026-08-18
tags: [xv6, riscv, syscall, trap, uart, vm]
status: notes
---

## 背景

在阅读 xv6-riscv 的 `user/usys.pl`、`kernel/trap.c`、`kernel/trampoline.S`、`kernel/vm.c` 等代码时产生一组问答，这里汇总记录。

## 提问与理解

### 1. 怎么生成系统调用汇编，供其他函数使用的啊？

`user/usys.pl` 不是汇编文件，而是一个 Perl 生成器。它根据脚本末尾的 `entry(...)` 调用，批量生成 `user/usys.S`。

Makefile 流程：

```make
$U/usys.S : $U/usys.pl
	perl $U/usys.pl > $U/usys.S

$U/usys.o : $U/usys.S
	$(CC) $(CFLAGS) -c -o $U/usys.o $U/usys.S
```

`usys.o` 属于用户库 `ULIB`，会被链接进所有用户程序：

```make
ULIB = $U/ulib.o $U/usys.o $U/printf.o $U/umalloc.o
```

每个系统调用都会生成一个薄薄的汇编桩：

```asm
.global fork
fork:
    li a7, SYS_fork
    ecall
    ret
```

用户 C 代码调用 `fork()` 时，参数已经在 `a0...a5`，桩只负责把系统调用号放入 `a7`，然后 `ecall` 陷入内核。内核处理完成后把返回值写回 `a0`，用户态再 `ret` 返回。

特例是 `sbrk`：汇编符号被命名为 `sys_sbrk`，真正的用户态 `sbrk()` 是 `ulib.c` 里的 C 包装函数，内部再调用 `sys_sbrk(n, SBRK_EAGER)`。

### 2. 用户态进程运行期间，来了 UART 中断，会发生什么？中断能在 user mode 执行？

结论：中断不能在 user mode 执行。UART 中断会让 CPU 从 U-mode 陷入 S-mode，由内核处理；用户进程只是被暂停并保存现场。

执行路径：

```text
UART 产生中断
  -> PLIC 向 hart 发送 S-mode 外部中断
  -> CPU 从用户态跳到 stvec 指向的 uservec
  -> uservec 保存用户寄存器
  -> 切换到内核栈
  -> 切换到内核页表
  -> jalr 到 usertrap()
  -> usertrap() 设置 stvec = kernelvec
  -> devintr() 识别 scause = 0x8000000000000009
  -> plic_claim() 得到 UART0_IRQ
  -> uartintr() 处理 UART
  -> plic_complete()
  -> 返回用户态
```

`uartintr()` 会：

- `ReadReg(ISR)` 清除/确认中断。
- 若 UART 发送完成，则 `tx_busy = 0` 并唤醒等待发送的线程。
- 若收到输入字符，则调用 `consoleintr()`，回显并写入 `cons.buf`，遇到换行时唤醒 `consoleread()`。

UART 中断不会触发 `yield()`，所以通常处理完后仍回到原来的用户进程继续执行。

为什么不能在 user mode 执行：

- 中断处理程序需要访问 PLIC、UART MMIO、内核数据结构和锁。
- 这些资源只在内核页表和 S-mode 下可访问。
- `stvec`、`scause`、`sstatus`、`sie`、`satp` 等 CSR 都是特权资源。

### 3. usertrap 这段代码映射到内核空间了吗？

是的。`usertrap` 是内核代码，地址在内核代码段中。

`kernel/kernel.sym` 中：

```text
0000000080002438 usertrap
```

内核页表建立时，把内核代码段做了 1:1 映射：

```c
kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext - KERNBASE, PTE_R | PTE_X);
```

所以 `usertrap` 在内核页表中可执行，虚拟地址等于物理地址。

但它不在用户进程页表中。用户页表只映射用户程序、`TRAMPOLINE` 和 `TRAPFRAME`。能跳到 `usertrap` 的原因是 `uservec` 先执行：

```asm
csrw satp, t1
jalr t0
```

即先切到内核页表，再跳转 `usertrap`。

### 4. 执行到 usertrap 开头时，中断是关着的吗？

是的，中断是关闭的。

这个关闭不是 C 代码主动做的，而是 RISC-V 硬件在发生 trap 时自动完成：

```text
SPIE = SIE
SIE  = 0
```

所以进入 `usertrap()` 时，`sstatus.SIE == 0`。

只有系统调用分支会主动开中断：

```c
if (r_scause() == 8) {
    ...
    p->trapframe->epc += 4;

    intr_on();
    syscall();
}
```

设备中断分支在 `devintr()` 处理期间仍保持关中断。之后 `prepare_return()` 会设置 `SPIE`，`sret` 返回用户态时恢复中断使能状态。

### 5. copyin 里的页边界计算是在做什么？

`kernel/vm.c` 的 `copyin()` 中：

```c
n = PGSIZE - (srcva - va0);
if (n > len)
  n = len;
```

这两行计算本轮最多能拷贝多少字节：

- `va0 = PGROUNDDOWN(srcva)` 得到当前页起始地址。
- `srcva - va0` 是页内偏移。
- `PGSIZE - (srcva - va0)` 是当前地址到页尾的剩余字节数。
- `if (n > len) n = len;` 保证不超过剩余需求。

本质是：

```text
n = min(到当前页末尾的字节数, 剩余需要拷贝的字节数)
```

例如 `PGSIZE=4096`、`srcva=0x1205`：

```text
va0 = 0x1000
offset = 0x205 = 517
到页尾 = 4096 - 517 = 3579
```

- 如果 `len=100`，则 `n=100`，只拷贝 100 字节。
- 如果 `len=10000`，则第一轮 `n=3579`，下一轮从 `0x2000` 继续。

详细内容另见 `copyin-page-boundary.md`。

## 要点

- 用户态系统调用通过 `usys.S` 里的汇编桩进入内核。
- UART 中断强制进入 S-mode，不在用户态执行。
- `usertrap` 映射在内核页表，但不在用户进程页表。
- 用户态 trap 进入 `usertrap` 时，硬件已经关闭中断。
- `copyin()` 逐页处理用户虚拟地址，避免一次跨多个物理页。

## 参考

- `/home/tzy/inspiration/my_xv6/user/usys.pl`
- `/home/tzy/inspiration/my_xv6/kernel/trap.c`
- `/home/tzy/inspiration/my_xv6/kernel/trampoline.S`
- `/home/tzy/inspiration/my_xv6/kernel/vm.c`
- `/home/tzy/inspiration/my_xv6/kernel/uart.c`
- `/home/tzy/inspiration/my_xv6/kernel/plic.c`
- `/home/tzy/inspiration/my_xv6/kernel/syscall.c`
