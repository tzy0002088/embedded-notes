---
title: "Trap 处理架构与内存屏障问答"
date: 2026-07-26
tags: [xv6, riscv, trap, stvec, fence, sfence_vma]
status: notes
---

## 背景

阅读 `kernel/trap.c`、`kernel/trampoline.S`、`kernel/kernelvec.S` 及 `kernel/vm.c` 中与 trap 入口、`stvec` 切换、内存屏障相关代码，产生一系列提问。

## 提问与理解

### 1. `uservec` 在哪被调用？—— 它不是被 call 的，是硬件按 stvec 跳的

`stvec` 寄存器存着 trap 入口地址。用户态发生任何 trap（ecall、中断、异常），硬件自动：`PC ← stvec`。`uservec` 就是被设在 `stvec` 里的那个地址。

谁设置？

```c
// usertrapret(): 准备返回用户态之前
uint64 trampoline_uservec = TRAMPOLINE + (uservec - trampoline);
w_stvec(trampoline_uservec);       // stvec = uservec (TRAMPOLINE 下的地址)
// 然后 sret → 回到用户态
```

为什么地址是 `TRAMPOLINE + (uservec - trampoline)` 而不是直接 `uservec`？因为 `uservec` 链接在内核 .text 段的物理地址附近，但用户页表中内核 .text 不可见。`TRAMPOLINE` 被映射到相同的虚拟地址（用户页表和内核页表都映射了同一物理页），所以用 TRAMPOLINE 下的地址做入口，用户态也能访问。

### 2. stvec 只有一个，S-mode 的 trap 怎么办？

`stvec` 的值在出入口被来回切换：

```c
// usertrap() — 刚从用户进来，切换到内核 trap 入口
w_stvec((uint64)kernelvec);

// usertrapret() — 准备返回用户，切回用户 trap 入口
w_stvec(trampoline_uservec);
```

```
U-mode 期间:   stvec = uservec     → trap → uservec → usertrap
S-mode 期间:   stvec = kernelvec   → trap → kernelvec → kerneltrap
```

不会出现"用户在跑但不走 uservec"或"内核在跑但不走 kernelvec"的情况——两个边界函数恰好卡在出入口。

### 3. stvec=uservec 时发生定时器中断？

完整路径：

```
用户代码跑着 (U-mode, stvec=uservec)
  → mtimer 中断
  → 硬件: SPP=0, scause=0x8000000000000005 (中断+STI), PC←uservec
  → uservec: 保存 32 个用户寄存器到 trapframe
  → 切内核页表 (w_satp)
  → 跳 usertrap()
  → usertrap: w_stvec(kernelvec)
  → devintr() 识别 scause→返回 2 (timer)
  → yield() → sched() → 切进程
```

`uservec` 不区分 trap 类型——它是个通用入口，不管 ecall/中断/异常，统一保存现场、切内核环境，然后交给 `usertrap` 根据 `scause` 分发。定时器中断在 xv6 中通过 `mideleg` 委托到了 S 模式，所以以 Supervisor Timer Interrupt 形式出现。

### 4. 为什么要拆 usertrap/kerneltrap，不能合并吗？

两种 trap 本质不同，合不到一起：

| | usertrap (SPP=0) | kerneltrap (SPP=1) |
|---|---|---|
| 入口 | uservec (TRAMPOLINE) | kernelvec (内核 .text) |
| 页表 | 用户页表 → 必须切 | 已在内核页表 |
| 栈 | 用户栈 → 必须切到内核栈 | 已在内核栈 |
| 保存暂存器 | 全部 32 个 | 只 caller-saved (~16 个) |
| intr_on()? | ✅ 可以开 | ❌ 不能开 |
| sleep/sched? | ✅ 可以 | ❌ 不能 |
| 本质 | 进程上下文 | 中断上下文 |

拆分原因：
- **入口汇编层就分岔了**：`uservec` 要切页表、切栈、保存 32 寄存器；`kernelvec` 只需 `sp-=256` + 存 16 个 caller-saved。不是在 C 代码里 `if (SPP)` 能统一的事。
- **后续策略完全不同**：用户态进来的 trap 可以 intr_on、可以 sched（有 proc 上下文）；中断进来的 trap 必须关中断、不能 sleep（没有 proc 上下文）。
- **入口地址要求不同**：`uservec` 必须在 TRAMPOLINE（用户可见），`kernelvec` 可以在内核 .text 任意处。

### 5. `sfence_vma()` — 一条指令两件事：TLB 刷新 + 页表写屏障

```c
void kvminithart() {
    sfence_vma();                          // 排干页表 store
    w_satp(MAKE_SATP(kernel_pagetable));   // 激活页表
    sfence_vma();                          // 刷旧 TLB
}
```

`sfence.vma zero, zero` 语义：
1. **内存屏障**：保证之前所有对页表内存的 store 在此之后全局可见
2. **TLB 刷新**：清空 TLB，后续地址翻译必须重新 walk 页表

两件事绑死是因为：只排 store 不刷 TLB → TLB 里还缓存旧翻译，不 walk 新页表；只刷 TLB 不排 store → walk 时从 cache 读到旧 PTE 值。

### 6. Store Buffer 是什么？

CPU 执行 `store` 指令时，不直接写 L1 cache——太慢。而是先扔进 CPU 内部的一个硬件小队列（store buffer），指令就算"完成"了，继续往下跑。数据从 buffer 异步排进 cache。

```
sd PTE[0], new  → [store buffer] → ... (延迟) ... → L1 D$ → RAM
```

问题：如果 buffer 没来得及排空，MMU page walker 可能从 cache 读到旧 PTE 值。`sfence.vma` 就是强制排干 buffer → cache，保证 MMU 下一次 walk 读到最新值。

### 7. `__atomic_thread_fence` 是保序，不是"排干 buffer"

```c
userinit();                                       // 各种初始化
__atomic_thread_fence(__ATOMIC_SEQ_CST);          // 保序
started = 1;                                      // 发信号
```

这条 fence 的**语义**是顺序约束：`fence` 之前的 store 不能被重排到 `fence` 之后。`fence` 之后的 store 不能被重排到 `fence` 之前。

硬件怎么实现这个约束（排 store buffer 是常见方式）是实现细节，但语义本身是**保序**。编译器和硬件两方面的重排都会被制止：

```
没有 fence:                        有 fence:
  started = 1  ──── 可能先出去      所有初始化 ─── fence ─── started = 1
  userinit()    ──── 可能后到       (保序，started 最后出去)
```

在多核场景下，CPU 0 的 fence（释放侧）和 CPU N 的 fence（获取侧）形成一对"门"：

```
CPU 0:  全部初始化 ── fence ── started = 1
CPU N:  while(started==0); ── fence ── 读初始化结果
```

释放侧保"写完再发信号"，获取侧保"收到信号再读"。两侧各锁一个方向。

### 8. `fence`、`sfence.vma`、`fence.i` 的分工

| 指令 | 管辖范围 | 保谁的序 |
|------|---------|---------|
| `fence rw,rw` | 普通内存 load/store | store vs store, store vs load（多核间） |
| `sfence.vma` | 页表内存 store | store(PTE) vs MMU page walk |
| `fence.i` | 指令内存 store | store(代码) vs ifetch（自修改代码） |

RISC-V 把三种屏障拆开，各管各的，分别对应数据、页表、指令三个领域。

### 9. TRAMPOLINE 的映射设计

RISC-V 硬件在 trap 时**不切换页表**，所以 `stvec` 指向的入口地址必须在**当前页表**（用户页表）中有效。

**映射在哪？** — 在 proc_pagetable() 中为每个用户进程映射，在 kvmmake() 中为内核页表映射：

```c
// proc.c: 用户页表 — 每个进程创建时
mappages(pagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X);

// vm.c: 内核页表 — 启动时
kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
```

同一物理页（trampoline.S 所在的页，地址 ≈ `0x80006000`），映射到同一虚拟地址 `TRAMPOLINE`（`MAXVA - PGSIZE = 0x3FFFFFFFF000`）。

`(uint64)trampoline` 是 trampoline.S 在内核 .text 段的链接地址（~`0x80006000`）。由于内核使用直接映射（VA=PA），它同时也是物理地址，作为 `mappages` 的物理页参数。

`trampoline:` 标签 == `uservec` 的地址（`uservec` 就在页起始处，`.align 4` 后紧跟 `uservec:`）。所以 `(uservec - trampoline) = 0`，`stvec` 值直接等于 `TRAMPOLINE`。

**为什么要切内核页表？** — 用户页表只映射了用户内存 + TRAMPOLINE + TRAPFRAME，没有内核代码、内核栈、proc 表、设备 MMIO。`uservec` 保存完用户寄存器后，必须切到内核页表才能调 C 代码 `usertrap()`、读 `struct proc`、访问设备等。

```
TRAMPOLINE: 用户/内核都能访问的最小共享区 → 入口代码 (uservec, userret)
TRAPFRAME:  用户/内核都能访问              → 寄存器保存区
切页表后:   内核独占                      → 所有其他工作
```

**链接脚本对 trampoline 的对齐约束**：

```ld
. = ALIGN(0x1000);
_trampoline = .;
*(trampsec)
. = ALIGN(0x1000);
ASSERT(. - _trampoline == 0x1000, "trampoline larger than one page");
```

保证 trampoline 代码不跨页，可用单个 4KB 页干净映射。

### 10. 每个进程都有独立的页表

```c
// proc.h
struct proc {
    pagetable_t pagetable;  // 每个进程独立的用户页表
};
```

`satp` 寄存器只有一个，进程切换时换值：

```c
// proc.c: scheduler()
w_satp(MAKE_SATP(p->pagetable));  // 换成目标进程的页表
sfence_vma();                       // 刷旧 TLB
swtch(&c->context, &p->context);    // 开始执行
```

```
CPU (一个 satp)                  内存中的页表 (每个进程一份)

satp ──→ proc A 页表            proc B 页表         proc C 页表
         (物理页 X)              (物理页 Y)           (物理页 Z)

         A 跑时 satp 指向 X → B 跑时 satp 指向 Y → C 跑时 satp 指向 Z
```

内核页表（`kernel_pagetable`）全局唯一一份。用户进程各有各的 `p->pagetable`。架构和 Linux/x86 的 per-process page table + CR3 切换完全一样。

### 11. 用户进程虚拟地址布局

```
MAXVA = 0x4000000000 (256 GB, Sv39)

  0x0000000000  ┌──────────────────┐
                │  text (代码段)    │  ← ELF LOAD segment (VA=0)
                ├──────────────────┤     uvmalloc + loadseg 拷入
                │  data + bss      │  ← ELF LOAD segment
                ├──────────────────┤
                │  heap            │  ← sbrk → growproc → uvmalloc
                │      ↓           │     向上增长
                │                  │
                │                  │     巨大空白区 (~256 GB)
                │                  │
                │      ↑           │
                │  user stack      │  ← exec → uvmalloc (1 页有效)
                ├──────────────────┤     USERSTACK = 1
                │  guard page      │  ← uvmclear (unmapped)
                │                  │
                │      ...         │     空白
                │                  │
  0x3FFFFFE000  ├──────────────────┤  ← TRAPFRAME
  0x3FFFFFF000  ├──────────────────┤  ← TRAMPOLINE (uservec/userret)
  0x4000000000  └──────────────────┘  ← MAXVA
```

创建过程 (exec.c: kexec):

```c
// ① 建空页表 (TRAMPOLINE + TRAPFRAME)
pagetable = proc_pagetable(p);

// ② 映射 ELF 的每个 LOAD 段 (text, data)
for (each ELF phdr) {
    uvmalloc(pagetable, ..., ph.vaddr, ph.vaddr + ph.memsz, PTE_R|PTE_W|PTE_X|PTE_U);
    loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz);
}

// ③ 映射用户栈 (1 页有效 + 1 页 guard)
uvmalloc(pagetable, sz, sz + (USERSTACK + 1) * PGSIZE, PTE_W);
uvmclear(pagetable, sz - (USERSTACK + 1) * PGSIZE);  // guard: 清权限
```

运行时 `sbrk` 扩堆：`sys_sbrk → growproc → uvmalloc(oldsz, newsz, PTE_W)`。

与 Linux 不同，xv6 没有 mmap、vDSO、ASLR，就是极简线性布局。

### 12. 内核全局变量也在内核页表里

`cpus[NCPU]`、`proc[NPROC]`、`initproc` 等全局变量编译后在 `.data` / `.bss` 段，落在 `etext` ~ `PHYSTOP` 之间。内核页表已通过 `kvmmake()` 中的直接映射全覆盖：

```c
kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP - (uint64)etext,
       PTE_R | PTE_W);   // etext ~ PHYSTOP, VA=PA
```

```
0x80000000  ┌──────────────┐
            │ kernel text  │  R|X
            ├──────────────┤ ← etext
            │ kernel data  │  ← cpus[], proc[], initproc 在这里
            │ kernel bss   │     R|W, VA=PA
            ├──────────────┤ ← end
            │ free memory  │  ← kalloc 管理
0x88000000  └──────────────┘ ← PHYSTOP
```

不需要单独为某个全局变量建映射——整段数据区一张大网全兜。

## 要点

- `uservec` 是硬件按 `stvec` 自动跳的，不是 `call` 的
- `stvec` 只有一个，但在 `usertrap()`/`usertrapret()` 边界被来回切换：U-mode→uservec，S-mode→kernelvec
- 定时器中断经过 `mideleg` 委托到 S 模式，用户态发生时走 `uservec → usertrap → devintr → yield`
- `usertrap`/`kerneltrap` 拆分是因为页表、栈、寄存器保存、中断策略都不同，汇编层已分岔
- `sfence.vma` = 页表 store 屏障 + TLB 刷新，一条指令绑死
- store buffer 是 CPU 写 cache 前的缓冲区，fence 通过排干它来保序
- `fence rw,rw` 保普通内存序，`sfence.vma` 保页表序，`fence.i` 保指令序
- fence 语义是保序，排 buffer 是硬件实现手段

## 参考

- `kernel/trap.c`: usertrap, kerneltrap, usertrapret, devintr
- `kernel/trampoline.S`: uservec, userret
- `kernel/kernelvec.S`: kernelvec
- `kernel/vm.c`: kvminithart
- `kernel/main.c`: 多核启动 fence
- `kernel/riscv.h`: sfence_vma
- RISC-V privileged spec: sfence.vma, stvec, scause
