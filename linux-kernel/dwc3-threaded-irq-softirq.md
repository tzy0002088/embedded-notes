---
title: "DWC3 threaded IRQ 为什么需要 local_bh_disable / softirq 上下文"
date: 2026-08-17
tags: [linux-kernel, usb, dwc3, interrupt, softirq, threaded-irq, riscv]
status: notes
---

## 背景

读 `drivers/usb/dwc3/gadget.c` 时，看到 `dwc3_thread_interrupt()` 里在
`spin_lock_irqsave()` 外面又包了一层 `local_bh_disable()` / `local_bh_enable()`：

```c
static irqreturn_t dwc3_thread_interrupt(int irq, void *_evt)
{
	struct dwc3_event_buffer *evt = _evt;
	struct dwc3 *dwc = evt->dwc;
	unsigned long flags;
	irqreturn_t ret = IRQ_NONE;

	local_bh_disable();
	spin_lock_irqsave(&dwc->lock, flags);
	ret = dwc3_process_event_buf(evt);
	spin_unlock_irqrestore(&dwc->lock, flags);
	local_bh_enable();

	return ret;
}
```

由此引出一串问题：为什么已经关了硬中断还要关 BH？softirq 是什么上下文？
它在什么时候执行？为什么自愿线程化的中断处理函数需要手动处理 softirq？

## 提问

1. 这里为什么还要加 `local_bh_disable()`？
2. `spin_lock_irqsave()` 只关本 CPU 硬中断，不关 softirq 吗？
3. softirq 是什么上下文？
4. 为什么说 softirq pending 后，threaded handler 返回时不会走 `irq_exit()`？
5. `dwc3_interrupt()` 退出后，才执行 `dwc3_thread_interrupt()` 吗？
6. 如果只有 `spin_lock_irqsave()`，持锁期间会被 softirq 重入吗？
7. `local_bh_enable()` 是唤醒还是同步执行 pending softirq？
8. 从进程上下文里同步调用 `do_softirq()`，执行 softirq 时还算进程上下文吗？
9. softirq 是直接使用当前进程的内核栈吗？在 RISC-V 上呢？

## 理解

### 1. DWC3 gadget 中断的注册方式

`dwc3_gadget_start()` 使用：

```c
request_threaded_irq(irq, dwc3_interrupt, dwc3_thread_interrupt,
		     IRQF_SHARED, "dwc3", dwc->ev_buf);
```

其中：

- `dwc3_interrupt()` 是 primary hard-IRQ handler；
- `dwc3_thread_interrupt()` 是 threaded handler；
- 这种写法没有 `IRQF_ONESHOT`，属于“自愿线程化”的中断处理。

基本流程是：

```text
dwc3_interrupt()                     // hardirq context
  -> dwc3_check_event_buf()
       -> 读事件计数
       -> 若有事件，缓存事件、设置 DWC3_EVENT_PENDING
       -> mask DWC3 event interrupt
       -> return IRQ_WAKE_THREAD

kernel IRQ core
  -> __irq_wake_thread()
  -> wake_up_process(irq thread)

dwc3_thread_interrupt()             // 进程上下文，由 IRQ thread 执行
  -> dwc3_process_event_buf()
       -> 处理事件
       -> completion 路径可能 raise softirq
       -> unmask DWC3 event interrupt
```

`dwc3_interrupt()` 先返回，内核再唤醒 IRQ 线程；`dwc3_thread_interrupt()`
不是被 hardirq 直接调用的，而是被异步调度执行。

### 2. 为什么需要 `local_bh_disable()`

官方提交是：

```text
usb: dwc3: gadget: Let the interrupt handler disable bottom halves.
commit 84918a89d6efaff075de570b55642b6f4ceeac6d
```

原因不是“持锁期间软中断会重入导致死锁”，而是：

- 自愿线程化的 threaded handler 运行时，IRQ core 不会像 forced-threaded
  handler 那样先 `local_bh_disable()`；
- gadget completion 路径，尤其是 network gadget，可能 raise
  `NET_RX_SOFTIRQ` / `NET_TX_SOFTIRQ`；
- 如果这个 softirq 只是被标记为 pending，threaded handler 返回时不会像
  hardirq 返回那样走 `irq_exit()`，因此 pending softirq 可能一直不被处理；
- 最坏情况下 CPU 进入 idle，NOHZ 发现还有 pending softirq，产生 warning。

所以 dwc3 显式写成：

```c
local_bh_disable();
spin_lock_irqsave(&dwc->lock, flags);
ret = dwc3_process_event_buf(evt);
spin_unlock_irqrestore(&dwc->lock, flags);
local_bh_enable();
```

目的是在释放锁后，通过 `local_bh_enable()` 给 pending softirq 制造一个
确定的执行点。

### 3. `spin_lock_irqsave()` 和 `local_bh_disable()` 管的是不同层

可以简化成：

```text
spin_lock_irqsave()
  = local_irq_save() + spin_lock()

local_irq_save() / local_irq_restore()
  -> 只操作本 CPU 的硬中断使能状态
```

`local_bh_disable()` 才是增加 `preempt_count` 中的 softirq 计数，
标记当前 CPU 上 BH/softirq 被禁用。

```text
spin_lock_irqsave() -> 管 hardirq 这一层
spin_lock_bh()      -> 管 BH/softirq 这一层
local_bh_disable()  -> 管 BH/softirq 这一层
```

### 4. softirq 是什么上下文

softirq 是 Linux 的延迟处理机制，运行在“软中断上下文”，属于中断上下文：

- 不是进程，没有自己的地址空间；
- 不能睡眠；
- 通常硬件中断是打开的，因此可以被 hardirq 打断；
- 同一个 CPU 上同一个 softirq 不会并发重入。

常见 softirq：

```text
HI_SOFTIRQ
TIMER_SOFTIRQ
NET_TX_SOFTIRQ
NET_RX_SOFTIRQ
BLOCK_SOFTIRQ
IRQ_POLL_SOFTIRQ
TASKLET_SOFTIRQ
SCHED_SOFTIRQ
HRTIMER_SOFTIRQ
RCU_SOFTIRQ
```

softirq 主要执行点：

1. 硬中断返回路径：`irq_exit()`；
2. `local_bh_enable()`；
3. `ksoftirqd` 内核线程。

### 5. “只有 spin_lock_irqsave 会被 softirq 重入吗？”

通常不会在持锁区间内重入：

- `spin_lock_irqsave()` 关闭本 CPU 硬中断；
- `spin_lock()` 关闭抢占；
- 因此当前 CPU 上不会发生硬中断返回路径的 softirq，也不会有 ksoftirqd
  被调度进来。

但 `spin_lock_irqsave()` 本身没有关闭 softirq，也没有清 pending。它只是
间接让持锁区间内通常不会执行 softirq。真正的问题是持锁期间 raise 的
softirq 会一直 pending，需要 `local_bh_enable()` 来及时处理。

### 6. `local_bh_enable()` 是同步执行 pending softirq

调用链大致是：

```text
dwc3_thread_interrupt()
  -> local_bh_enable()
      -> __local_bh_enable_ip()
          -> do_softirq()
              -> __do_softirq()
                  -> h->action()
```

在 dwc3 这个调用点，`local_bh_enable()` 是在
`spin_unlock_irqrestore()` 之后调用的，当前已恢复抢占，且是进程上下文，
因此如果 `local_softirq_pending()` 非空，会直接 `do_softirq()`，不是
简单地去唤醒 `ksoftirqd`。

只有当当前上下文不适合直接执行 softirq，例如不可抢占，或者 softirq 压力
过大、处理时间超限时，内核才把剩余部分交给 `ksoftirqd`。

### 7. 从进程上下文同步调用 softirq，为什么不算进程上下文

执行到 softirq handler 时，内核会通过 `preempt_count` 设置 softirq 位，
所以 `in_softirq()` / `in_interrupt()` 返回真。

上下文分类不取决于“当前有没有 task_struct”，而是取决于 `preempt_count`
中的中断相关位：

```text
进程上下文：hardirq/softirq 位通常为 0
硬中断上下文：hardirq 位被设置
软中断上下文：softirq 位被设置
```

因此：

```text
dwc3_thread_interrupt()          -> 进程上下文
local_bh_enable()                -> 进程上下文发起
do_softirq() 执行 h->action()    -> softirq 上下文
h->action() 返回后               -> 回到原来的进程上下文
```

所以 softirq handler 即使是被某个线程同步调用的，执行期间仍然不能睡眠。

### 8. softirq 使用哪个栈

`do_softirq()` 会调用 `do_softirq_own_stack()`。是否使用独立栈取决于架构：

```c
// include/asm-generic/softirq_stack.h
static inline void do_softirq_own_stack(void)
{
    __do_softirq();
}
```

在没有 `CONFIG_SOFTIRQ_ON_OWN_STACK` 的架构上，`do_softirq_own_stack()`
就是直接执行 `__do_softirq()`，因此 softirq 使用当前内核栈。

在 x86_64、arm64、RISC-V 等默认开启独立 IRQ/softirq stack 的架构上，
`do_softirq_own_stack()` 会切换到 per-CPU IRQ/softirq stack。

### 9. RISC-V 上 softirq 的栈路径

RISC-V 默认 `CONFIG_IRQ_STACKS=y`，并选择：

```kconfig
select HAVE_IRQ_EXIT_ON_IRQ_STACK
select HAVE_SOFTIRQ_ON_OWN_STACK
```

RISC-V 的实现：

```c
// arch/riscv/kernel/irq.c
void do_softirq_own_stack(void)
{
	if (on_thread_stack())
		call_on_irq_stack(NULL, ___do_softirq);
	else
		__do_softirq();
}
```

从 `dwc3_thread_interrupt()` 这样的线程上下文调用时，`on_thread_stack()`
为真，所以会：

```text
do_softirq()
  -> do_softirq_own_stack()
      -> call_on_irq_stack()
          -> 切换 sp 到 per-CPU IRQ stack
          -> __do_softirq()
          -> 切回原 thread stack
```

因此 RISC-V 默认配置下，softirq 不是直接使用当前 IRQ 线程的内核栈，而是
使用独立的 per-CPU IRQ/softirq stack。

### 10. 最终可复述的总结

这一串问题收敛成三句话：

```text
1. request_threaded_irq() 注册的 primary handler 先运行；
   它返回 IRQ_WAKE_THREAD 后，内核唤醒 IRQ 线程，
   dwc3_thread_interrupt() 在之后某个调度点运行。

2. local_bh_disable() 本身不查 softirq，它是关闭 BH 的标记；
   真正主动检查并执行 pending softirq 的是 local_bh_enable()。

3. 在当前 RISC-V 默认配置下，从 dwc3_thread_interrupt() 进入
   do_softirq() 时，会切到独立的 per-CPU IRQ/softirq stack，
   但 current 仍然是原来的 dwc3 IRQ 线程。
```

更准确地说，`local_bh_disable()` / `local_bh_enable()` 是一对组合：

```c
local_bh_disable();                     // 暂时关闭本 CPU 的 BH/softirq
spin_lock_irqsave(&dwc->lock, flags);   // 关闭硬中断并持锁
ret = dwc3_process_event_buf(evt);      // 期间可能 raise softirq
spin_unlock_irqrestore(&dwc->lock, flags);
local_bh_enable();                      // 检查并执行 pending softirq
```

核心动机不是“防 softirq 持锁重入”，而是给 threaded handler 里产生的
pending softirq 一个确定的执行点，避免它被推迟到下一次随机硬中断。

## 要点

- `dwc3_thread_interrupt()` 是自愿线程化的 threaded handler，不是被
  `dwc3_interrupt()` 直接调用，而是由内核 IRQ thread 异步执行。
- `spin_lock_irqsave()` 只关本 CPU 硬中断，不关闭 softirq。
- softirq 是中断上下文，不能睡眠；常见执行点是 `irq_exit()`、
  `local_bh_enable()` 和 `ksoftirqd`。
- dwc3 里 `local_bh_disable()/local_bh_enable()` 的核心目的，是让处理
  gadget 事件期间 raise 的 softirq 在释放锁后被及时执行，而不是一直
  pending 到下一次随机硬中断。
- `local_bh_enable()` 在适合直接执行的进程上下文里会同步调用
  `do_softirq()`；只有不适合直接执行时才交给 `ksoftirqd`。
- softirq 执行期间虽然 `current` 未变，但 `preempt_count` 已标记为
  softirq 上下文，因此不是普通进程上下文。
- RISC-V 默认启用独立 IRQ/softirq stack，`do_softirq_own_stack()` 会切到
  per-CPU IRQ stack。
- `local_bh_disable()` 只关闭 BH，`local_bh_enable()` 才负责检查并执行
  pending softirq；不要只归因到 `local_bh_disable()` 上。

## 参考

- `drivers/usb/dwc3/gadget.c`
- `kernel/irq/handle.c`
- `kernel/irq/manage.c`
- `kernel/softirq.c`
- `include/linux/bottom_half.h`
- `arch/riscv/kernel/irq.c`
- `arch/riscv/kernel/entry.S`
- Linux 提交：`84918a89d6efaff075de570b55642b6f4ceeac6d`
  `usb: dwc3: gadget: Let the interrupt handler disable bottom halves.`
