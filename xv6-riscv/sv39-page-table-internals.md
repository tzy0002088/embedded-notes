---
title: "Sv39 页表深入、内存映射与缺页处理问答"
date: 2026-07-21
tags: [xv6, riscv, sv39, page-table, mmu, trap, stack]
status: notes
---

## 背景

逐行阅读 `kernel/vm.c` 和 `kernel/proc.c`，对 Sv39 页表项结构、多级映射过程、内核栈布局、以及用户态/内核态缺页的不同处理方式产生一系列提问。

## 提问与理解

### 1. PTE 完整的 64 位字段

RISC-V Sv39 PTE 是 64 位：

```
 63..54   53..10      9..8   7    6    5    4    3    2    1    0
┌────────┬────────────┬──────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 保留=0 │ PPN[53:10] │ RSW  │ D  │ A  │ G  │ U  │ X  │ W  │ R  │ V  │
└────────┴────────────┴──────┴────┴────┴────┴────┴────┴────┴────┴────┘
          └── PTE2PA/PA2PTE ──┘     └────────── PTE_FLAGS ────────────┘
```

| Bit | 宏 | 含义 |
|-----|-----|------|
| 0 | `PTE_V` | Valid — 此项有效 |
| 1 | `PTE_R` | Readable |
| 2 | `PTE_W` | Writable |
| 3 | `PTE_X` | Executable |
| 4 | `PTE_U` | User — U=1 用户态可访问；U=0 只有 S/M 能访问 |
| 5 | (G) | Global — 切换地址空间不刷 TLB。xv6 未使用 |
| 6 | (A) | Accessed — 硬件自动置 1（需开启 Svadu/ADUE） |
| 7 | (D) | Dirty — 硬件自动置 1（需开启 Svadu/ADUE） |
| 8-9 | (RSW) | Reserved for SW — 给 OS 随意用。xv6 未使用 |
| 10-53 | PPN | 物理页号，44 bit |

R/W/X 全 0 且 V=1 → 非叶子 PTE（指向下一级页表）。至少一位 R/W/X=1 → 叶子 PTE。

xv6 中 A/D 位由硬件自动管理（`MENVCFG_ADUE` 已开启），不需要在 trap handler 中处理 A/D page fault。

### 2. "叶子"的意思——非叶子 vs 叶子 PTE

```
Level 2 (根)        Level 1             Level 0           物理页
┌──────────┐      ┌──────────┐        ┌──────────┐      ┌──────┐
│ PTE[0]   │──→   │ PTE[0]   │──→     │ PTE[0]   │──→   │ 4KB  │
│ R=W=X=0  │      │ R=W=X=0  │        │ R=1,W=1  │      │ 数据  │
│ V=1      │      │ V=1      │        │ V=1      │      └──────┘
└──────────┘      └──────────┘        └──────────┘
   非叶子            非叶子               叶子 ↑
   继续往下          继续往下             到达物理页
```

非叶子 PTE (R/W/X 全为 0)：只做路由，指向下一级页表页。叶子 PTE (R/W/X 不全为 0)：指向最终物理页，停下。

### 3. 一个 entry 能映射多大？—— Sv39 的三级叶子

RISC-V Sv39 支持在不同层级停下形成"大页"：

```
停止在哪      用了几层索引    剩下做偏移(bit)    映射大小    物理对齐
────────      ────────────    ──────────────    ────────    ────────
Level 2 叶子   9 bit (L2)     30 bit            1 GB        1 GB
Level 1 叶子   18 bit (L2+L1)  21 bit            2 MB        2 MB
Level 0 叶子   27 bit (全三层)  12 bit            4 KB        4 KB
```

越早停，剩下的位全当偏移 → 一个 entry 覆盖的范围越大。但物理页必须对齐到对应大小。

Level 1 叶子 (2MB)：PPN[2] 和 PPN[1] 做基址，PPN[0] 必须为 0（2MB 对齐）。释放出来 L0 idx 的 9 位加入偏移 → 21 位 = 2MB。

Level 2 叶子 (1GB)：只用 PPN[2] 做基址，PPN[1] 和 PPN[0] 必须为 0（1GB 对齐）。释放 L1+L0 idx 的 18 位加入偏移 → 30 位 = 1GB。

xv6 只用 Level 0 叶子（4KB），不支持大页：`walk()` 无条件走到底返回 `&pagetable[PX(0, va)]`。

### 4. PLIC 64MB 映射完整过程

```c
kvmmap(kpgtbl, PLIC, PLIC, 0x4000000, PTE_R | PTE_W);
//             0x0C000000  64MB
```

Sv39 切片（起始 VA = `0x0C000000`）：L2 idx=0, L1 idx=96, L0 idx=0。

`mappages` 循环 16384 次 (64MB ÷ 4KB)，每次调用 `walk(pagetable, a, 1)`：

- **Level 2**：`kpgtbl[0]` — PLIC 之前已由 UART/VIRTIO 映射触发分配好 L1 页表，V=1，直接复用。
- **Level 1**：`L1[96]` — 第一页时 V=0 → `kalloc()` 分配 L0 页表 #1，`L1[96] = PA2PTE | PTE_V`。后续 511 页复用同一 L0 表。
  - 到 VA=`0x0C200000` 时 `PX(1,va)=97` — 新的 L1 entry → 再 `kalloc` L0 页表 #2。
  - 依此类推，L1 用到 `[96]~[127]` 共 32 个 entry，分配 32 张 L0 页表。
- **Level 0**：每张 L0 表 512 项，每项映射 4KB。32 张 × 512 项 = 16384 个叶子 PTE。

最终结构：

| 层级 | 页数 | PTE 使用 | 每项管多大 |
|------|------|----------|-----------|
| L2 | 1 (kvmmake 时) | `[0]` 1 个 | 512 GB |
| L1 | 1 页 | `[96]~[127]` 32 个 | 2 MB |
| L0 | 32 页 | 全部 512×32=16384 个 | **4 KB** |

核心逻辑：`walk()` 判断"这一级有这个 PTE 吗？V=1 就复用，V=0 且 alloc=1 就 kalloc 分配"。

### 5. `proc_mapstacks` — 预分配所有内核栈

```c
for (p = proc; p < &proc[NPROC]; p++) {         // 遍历 64 个进程槽
    char *pa = kalloc();                          // 分配一页物理内存
    uint64 va = KSTACK((int)(p - proc));          // 计算内核栈 VA
    kvmmap(kpgtbl, va, (uint64)pa, PGSIZE, ...); // 映射到内核页表
}
```

64 个进程槽 × 4KB = 256KB，启动时一次性分配。每个进程独享一块内核栈，处理系统调用/中断时用。

内核栈布局（紧贴 TRAMPOLINE 下方）：

```
KSTACK(p) = TRAMPOLINE - ((p) + 1) × 2 × PGSIZE

TRAMPOLINE (MAXVA - PGSIZE)  ← trampoline 页
  ↓ guard page (unmapped)    ← 隔离
KSTACK(0)                    ← proc[0] 内核栈 (4KB, R|W)
  ↓ guard page (unmapped)    ← 溢出保护
KSTACK(1)                    ← proc[1] 内核栈
  ↓ guard page
...
KSTACK(63)                   ← proc[63] 内核栈
```

每个栈占 2 页 (1 有效 + 1 guard)。栈向下增长，溢出踩到 guard page → page fault → panic。

### 6. guard page — 内核态 vs 用户态缺页

**内核态访问 unmapped guard page**：

```
kerneltrap():
  devintr() == 0                    // 不认识 page fault
  → panic("kerneltrap")             // 直接停机
```

内核态缺页 = bug（栈溢出、空指针），无法恢复，直接 panic。`stval` 寄存器记录犯规 VA 方便排查。

**用户态 page fault（scause=13/15）**：完全不同——这是正常的"延迟分配"机制。

```c
// usertrap():
} else if ((r_scause() == 15 || r_scause() == 13) &&
           vmfault(p->pagetable, r_stval(), ...) != 0) {
    // page fault on lazily-allocated page — 正常，已修复
}
```

流程：`sbrk(n)` 只增加 `p->sz` 不分配物理页 → 用户后来访问该地址 → MMU 报 page fault → `vmfault()` 现场 `kalloc`+`mappages`+返回 → 用户指令重试，这次 PTE_V=1，成功。

```
用户态缺页                        内核态缺页
───────                          ───────
vmfault() 尝试修复                panic() 直接死
→ 分配物理页                      → 这不是延迟分配
→ 映射                            → 是真 bug
→ 返回重试                        → guard page / NULL deref
```

### 7. 用户态 vs 内核态缺页完整对比

| | 用户态 page fault | 内核态 page fault |
|---|---|---|
| 触发 | 延迟分配、COW、swap | 栈溢出 guard page、空指针 |
| handler | `usertrap()` → `vmfault()` | `kerneltrap()` → `panic()` |
| 行为 | 分配页 + 映射 + 返回重试 | 打印诊断 + 停机 |
| 能恢复? | ✅ | ❌ |

### 8. ecall 之后能 sleep 吗？—— 进程上下文 vs 中断上下文 。这里待定，说的不清楚

**硬件层面**：ecall 和硬件中断走同一个入口（`stvec`），都是 trap，硬件不区分。

**软件层面**：`usertrap()` 根据 `scause` 区分：

```c
if (r_scause() == 8) {        // ecall — 同步，来自当前进程
    intr_on();                 // 开中断！后续可以 sleep
    syscall();                 // 进程上下文
}
else if (devintr()) {         // 硬件中断 — 异步
    // 中断关着，快速处理，不能 sleep
}
```

```
ecall / page fault (同步):           硬件中断 (异步):
  当前指令引发的                     任何时候都可能到达
  myproc() 可靠 → 有主人             myproc() 不可靠 → 没有主人
  ✅ 能 sleep / schedule()           ❌ 不能 sleep
  ✅ 能 intr_on() 开中断             ❌ 必须关中断快速处理
```

**Linux 同理**：

```c
// x86: entry_SYSCALL_64
sti                    // 开中断
call do_syscall_64     // 进程上下文，可以 schedule
```

**为什么能 schedule？**：`intr_on()` 开了中断，后续就是普通进程上下文，有私有内核栈和 `struct proc`。`swtch()` 把 callee-saved 寄存器保存到 `p->context`，切调度器栈→`ret` 到 `scheduler()`。整个调用链（usertrap→syscall→sched）冻结在内核栈上，之后被调度回来从 `swtch` 下一行继续执行，一路返回到 `sret`。不是"在中断 handler 里 sleep"，而是"开中断后，这条执行路径本身就变成了进程上下文"。

## 要点

- PTE 低 10 位是标志 (V/R/W/X/U/A/D/RSW)，高 44 位是 PPN
- 非叶子 PTE (R/W/X=0) 只做路由，叶子 PTE (R/W/X 不全 0) 指向物理页
- Sv39 支持 4KB/2MB/1GB 三级叶子，xv6 只用 4KB
- PLIC 64MB = 16384 个 4KB 页，通过 32 张 L0 页表映射
- `proc_mapstacks` 预分配 64 个内核栈 (×4KB=256KB)，每栈配一个 unmapped guard page
- 内核态缺页 → panic；用户态缺页 → vmfault 延迟分配
- ecall 进来后 `intr_on()` → 进程上下文 → 可以 sleep/schedule；中断 handler 不能

## 参考

- `kernel/vm.c`: walk, mappages, kvmmap, kvmmake, vmfault
- `kernel/proc.c`: proc_mapstacks
- `kernel/trap.c`: usertrap, kerneltrap
- `kernel/memlayout.h`: KSTACK, TRAMPOLINE, 内存布局
- `kernel/riscv.h`: PTE_V/R/W/X/U, PA2PTE, PTE2PA
- RISC-V privileged spec, Sv39 chapter
