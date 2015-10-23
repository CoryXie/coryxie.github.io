---
layout: post
title: "Linux Kernel HARDIRQ and SOFTIRQ"
description:
headline:
modified: 2015-10-06
category: KernelDev
tags: [Kernel]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

Linux kernel has three contexts of thread executions: HARDIRQ context, SOFTIRQ context, and PROCESS context. We can verify that we are in interrupt context by involving `in_interrupt()`, when it returns none-zero, it indicates we are working in hardirq or softirq. It is necessary to take the time to further explore the difference between the hardirq and softirq. 

## Overal Interrupt Handling Process

This section describes the overall interrupt handling framework. We could see that interrupt handling is generally divided into two parts, HARDIRQ and SOFTIRQ. 

For ARM/ARM64 with GICv3, the `linux-4.2/drivers/irqchip/irq-gic-v3.c` code initialize the system interrupt handing in the function `gic_of_init()`, where it calls `set_handle_irq(gic_handle_irq)`. This code setups the `gic_handle_irq()` to be called by the assembly code in `linux-4.2/arch/arm64/kernel/entry.S`: 

```c

	/*
	 * Interrupt handling.
	 */
	.macro	irq_handler
	adrp	x1, handle_arch_irq
	ldr	x1, [x1, #:lo12:handle_arch_irq]
	mov	x0, sp
	blr	x1
	.endm

		.align	6
	el1_irq:
		kernel_entry 1
		enable_dbg
	#ifdef CONFIG_TRACE_IRQFLAGS
		bl	trace_hardirqs_off
	#endif
	
		irq_handler
	
	#ifdef CONFIG_PREEMPT
		get_thread_info tsk
		ldr	w24, [tsk, #TI_PREEMPT]		// get preempt count
		cbnz	w24, 1f				// preempt count != 0
		ldr	x0, [tsk, #TI_FLAGS]		// get flags
		tbz	x0, #TIF_NEED_RESCHED, 1f	// needs rescheduling?
		bl	el1_preempt
	1:
	#endif
	#ifdef CONFIG_TRACE_IRQFLAGS
		bl	trace_hardirqs_on
	#endif
		kernel_exit 1
	ENDPROC(el1_irq)
	
	#ifdef CONFIG_PREEMPT
	el1_preempt:
		mov	x24, lr
	1:	bl	preempt_schedule_irq		// irq en/disable is done inside
		ldr	x0, [tsk, #TI_FLAGS]		// get new tasks TI_FLAGS
		tbnz	x0, #TIF_NEED_RESCHED, 1b	// needs rescheduling?
		ret	x24
	#endif

```

So we can see that the `handle_arch_irq()` is in fact `gic_handle_irq()`, and it is called with interrupts disabled.

```c

	static asmlinkage void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
	{
		u64 irqnr;
	
		do {
			irqnr = gic_read_iar();
	
			if (likely(irqnr > 15 && irqnr < 1020) || irqnr >= 8192) {
				int err;
				err = handle_domain_irq(gic_data.domain, irqnr, regs);
				if (err) {
					WARN_ONCE(true, "Unexpected interrupt received!\n");
					gic_write_eoir(irqnr);
				}
				continue;
			}
			if (irqnr < 16) {
				gic_write_eoir(irqnr);
	#ifdef CONFIG_SMP
				handle_IPI(irqnr, regs);
	#else
				WARN_ONCE(true, "Unexpected SGI received!\n");
	#endif
				continue;
			}
		} while (irqnr != ICC_IAR1_EL1_SPURIOUS);
	}

```

The `handle_domain_irq()` is the centrol function to handle an interrupt recorded in the GIC interrupt controller, and it directly calls `__handle_domain_irq()`, which is defined in `linux-4.2/kernel/irq/irqdesc.c`:

```c

	#ifdef CONFIG_HANDLE_DOMAIN_IRQ
	/**
	 * __handle_domain_irq - Invoke the handler for a HW irq belonging to a domain
	 * @domain:	The domain where to perform the lookup
	 * @hwirq:	The HW irq number to convert to a logical one
	 * @lookup:	Whether to perform the domain lookup or not
	 * @regs:	Register file coming from the low-level handling code
	 *
	 * Returns:	0 on success, or -EINVAL if conversion has failed
	 */
	int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
				bool lookup, struct pt_regs *regs)
	{
		struct pt_regs *old_regs = set_irq_regs(regs);
		unsigned int irq = hwirq;
		int ret = 0;
	
		irq_enter();
	
	#ifdef CONFIG_IRQ_DOMAIN
		if (lookup)
			irq = irq_find_mapping(domain, hwirq);
	#endif
	
		/*
		 * Some hardware gives randomly wrong interrupts.  Rather
		 * than crashing, do something sensible.
		 */
		if (unlikely(!irq || irq >= nr_irqs)) {
			ack_bad_irq(irq);
			ret = -EINVAL;
		} else {
			generic_handle_irq(irq);
		}
	
		irq_exit();
		set_irq_regs(old_regs);
		return ret;
	}
	#endif

```

## HARDIRQ stage

In the `__handle_domain_irq()` function, calling `irq_enter()` can be seen as the beginning of hardirq, and calling the function `irq_exit()` marks beginning of softirq. The `irq_enter()` function call is the core of `__irq_enter()`, the main effect of the latter is to increase (+1) HARDIRQ portion of variable `preempt_count`, i.e. it identifies a hardirq context, so it is considered that the `gic_handle_irq()` functions calling `irq_enter()` starts the hardirq stage. The following is `irq_enter()` as defined in `linux-4.2/kernel/softirq.c`:

```c

	/*
	 * Enter an interrupt context.
	 */
	void irq_enter(void)
	{
		rcu_irq_enter();
		if (is_idle_task(current) && !in_interrupt()) {
			/*
			 * Prevent raise_softirq from needlessly waking up ksoftirqd
			 * here, as softirq will be serviced on return from interrupt.
			 */
			local_bh_disable();
			tick_irq_enter();
			_local_bh_enable();
		}
	
		__irq_enter();
	}

```

The `__irq_enter()` is defined in `linux-4.2/include/linux/hardirq.h`:

```c

	/*
	 * It is safe to do non-atomic ops on ->hardirq_context,
	 * because NMI handlers may not preempt and the ops are
	 * always balanced, so the interrupted value of ->hardirq_context
	 * will always be restored.
	 */
	#define __irq_enter()					\
		do {						\
			account_irq_enter_time(current);	\
			preempt_count_add(HARDIRQ_OFFSET);	\
			trace_hardirq_enter();			\
		} while (0)

```

The `HARDIRQ_OFFSET` is defined in `linux-4.2/include/linux/preempt.h`, which:

```c

	/*
	 * We put the hardirq and softirq counter into the preemption
	 * counter. The bitmask has the following meaning:
	 *
	 * - bits 0-7 are the preemption count (max preemption depth: 256)
	 * - bits 8-15 are the softirq count (max # of softirqs: 256)
	 *
	 * The hardirq count could in theory be the same as the number of
	 * interrupts in the system, but we run all interrupt handlers with
	 * interrupts disabled, so we cannot have nesting interrupts. Though
	 * there are a few palaeontologic drivers which reenable interrupts in
	 * the handler, so we need more than one bit here.
	 *
	 *         PREEMPT_MASK:	0x000000ff
	 *         SOFTIRQ_MASK:	0x0000ff00
	 *         HARDIRQ_MASK:	0x000f0000
	 *             NMI_MASK:	0x00100000
	 *       PREEMPT_ACTIVE:	0x00200000
	 * PREEMPT_NEED_RESCHED:	0x80000000
	 */
	#define PREEMPT_BITS	8
	#define SOFTIRQ_BITS	8
	#define HARDIRQ_BITS	4
	#define NMI_BITS	1
	
	#define PREEMPT_SHIFT	0
	#define SOFTIRQ_SHIFT	(PREEMPT_SHIFT + PREEMPT_BITS)
	#define HARDIRQ_SHIFT	(SOFTIRQ_SHIFT + SOFTIRQ_BITS)
	#define NMI_SHIFT	(HARDIRQ_SHIFT + HARDIRQ_BITS)
	
	#define __IRQ_MASK(x)	((1UL << (x))-1)
	
	#define PREEMPT_MASK	(__IRQ_MASK(PREEMPT_BITS) << PREEMPT_SHIFT)
	#define SOFTIRQ_MASK	(__IRQ_MASK(SOFTIRQ_BITS) << SOFTIRQ_SHIFT)
	#define HARDIRQ_MASK	(__IRQ_MASK(HARDIRQ_BITS) << HARDIRQ_SHIFT)
	#define NMI_MASK	(__IRQ_MASK(NMI_BITS)     << NMI_SHIFT)
	
	#define PREEMPT_OFFSET	(1UL << PREEMPT_SHIFT)
	#define SOFTIRQ_OFFSET	(1UL << SOFTIRQ_SHIFT)
	#define HARDIRQ_OFFSET	(1UL << HARDIRQ_SHIFT)
	#define NMI_OFFSET	(1UL << NMI_SHIFT)
	
	#define SOFTIRQ_DISABLE_OFFSET	(2 * SOFTIRQ_OFFSET)

```
The ability to respond to external interrupts for the processor at this time remains disabled, because ISR is essentially controlled by the device drivers, the kernel can not guarantee whether the interrupt will be enabled or not in the ISR of the device driver. 

According to this core design philosophy, during the hardirq stage it is better not to enable the interrupt, so in the context of hardirq, device driver ISR should return as soon as possible (complete only the most critical tasks). The remaining work can be left to the softirq stage, where the kernel will have external interrupts enabled, it also means that you can always be interrupted by an external interrupt in softirq context. 


## SOFTIRQ stage

Here we focus on the implementation of the softirq stage. The `gic_handle_irq()` function call `irq_exit()` to start the softirq stage. The `irq_exit()` function will decrease (-1) the HARDIRQ portion in `preempt_count` variable, the aim is to remove the flag of hardirq context. Then it has a critical call to `invoke_softirq()` for the actual processing of softirq: 

```c

	/*
	 * Exit an interrupt context. Process softirqs if needed and possible:
	 */
	void irq_exit(void)
	{
	#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
		local_irq_disable();
	#else
		WARN_ON_ONCE(!irqs_disabled());
	#endif
	
		account_irq_exit_time(current);
		preempt_count_sub(HARDIRQ_OFFSET);
		if (!in_interrupt() && local_softirq_pending())
			invoke_softirq();
	
		tick_irq_exit();
		rcu_irq_exit();
		trace_hardirq_exit(); /* must be last! */
	}

```

The main intention of `in_interrupt()` function is based on the current `preempt_count` variable to determine whether the current code is executed in an interrupt context. 

```c

	#define hardirq_count()	(preempt_count() & HARDIRQ_MASK)
	#define softirq_count()	(preempt_count() & SOFTIRQ_MASK)
	#define irq_count()	(preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK \
					 | NMI_MASK))
	
	/*
	 * Are we doing bottom half or hardware interrupt processing?
	 * Are we in a softirq context? Interrupt context?
	 * in_softirq - Are we currently processing softirq or have bh disabled?
	 * in_serving_softirq - Are we currently processing softirq?
	 */
	#define in_irq()		(hardirq_count())
	#define in_softirq()		(softirq_count())
	#define in_interrupt()		(irq_count())
	#define in_serving_softirq()	(softirq_count() & SOFTIRQ_OFFSET)
	
	/*
	 * Are we in NMI context?
	 */
	#define in_nmi()	(preempt_count() & NMI_MASK)
	
```
According to the definition `in_interrupt()`, the Linux kernel considers HARDIRQ, SOFTIRQ, and NMI belonging to interrupt context, so whether softirq stage is executed depends on: 

>* 1) is currently in interrupt context
>* 2) whether there is pending softirq needs to be addressed

The first condition in `irq_exit()`, i.e `!in_interrupt()`, is designed to prevent the re-entry of softirq handling. Because once any pending softirq needs treatment, the `invoke_softirq()` call (the actual work is done in `__do_softirq()` function) will first increase (+1) the SOFTIRQ part of the `preempt_count` variable. If the process is currently doing softirq when external interrupt occurs, and the hardirq stage identifies a pending softirq, it will return directly in `irq_exit()` function, and will not call into `invoke_softirq()`. 

The softirq code `do_softirq()` works as follows in `linux-4.2/kernel/softirq.c`: 

```c

	static inline void invoke_softirq(void)
	{
		if (!force_irqthreads) {
	#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
			/*
			 * We can safely execute softirq on the current stack if
			 * it is the irq stack, because it should be near empty
			 * at this stage.
			 */
			__do_softirq();
	#else
			/*
			 * Otherwise, irq_exit() is called on the task stack that can
			 * be potentially deep already. So call softirq in its own stack
			 * to prevent from any overrun.
			 */
			do_softirq_own_stack();
	#endif
		} else {
			wakeup_softirqd();
		}
	}

	asmlinkage __visible void __do_softirq(void)
	{
		unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
		unsigned long old_flags = current->flags;
		int max_restart = MAX_SOFTIRQ_RESTART;
		struct softirq_action *h;
		bool in_hardirq;
		__u32 pending;
		int softirq_bit;
	
		/*
		 * Mask out PF_MEMALLOC s current task context is borrowed for the
		 * softirq. A softirq handled such as network RX might set PF_MEMALLOC
		 * again if the socket is related to swap
		 */
		current->flags &= ~PF_MEMALLOC;
	
		pending = local_softirq_pending();
		account_irq_enter_time(current);
	
		__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
		in_hardirq = lockdep_softirq_start();
	
	restart:
		/* Reset the pending bitmask before enabling irqs */
		set_softirq_pending(0);
	
		local_irq_enable();
	
		h = softirq_vec;
	
		while ((softirq_bit = ffs(pending))) {
			unsigned int vec_nr;
			int prev_count;
	
			h += softirq_bit - 1;
	
			vec_nr = h - softirq_vec;
			prev_count = preempt_count();
	
			kstat_incr_softirqs_this_cpu(vec_nr);
	
			trace_softirq_entry(vec_nr);
			h->action(h);
			trace_softirq_exit(vec_nr);
			if (unlikely(prev_count != preempt_count())) {
				pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
				       vec_nr, softirq_to_name[vec_nr], h->action,
				       prev_count, preempt_count());
				preempt_count_set(prev_count);
			}
			h++;
			pending >>= softirq_bit;
		}
	
		rcu_bh_qs();
		local_irq_disable();
	
		pending = local_softirq_pending();
		if (pending) {
			if (time_before(jiffies, end) && !need_resched() &&
			    --max_restart)
				goto restart;
	
			wakeup_softirqd();
		}
	
		lockdep_softirq_end(in_hardirq);
		account_irq_exit_time(current);
		__local_bh_enable(SOFTIRQ_OFFSET);
		WARN_ON_ONCE(in_interrupt());
		tsk_restore_flags(current, old_flags, PF_MEMALLOC);
	}
	
```

The processing for each type of softirq occurs in the above `while()` loop in `__do_softirq()`, the principle is very simple, as demostrated in the following figure: 

<img src="{{ site.baseurl }}/images/2015-10-06-1/local_softirq_pending.png" alt="Kernel local_softirq_pending() macro">

```c
	
	#define NR_IPI	5
	
	typedef struct {
		unsigned int __softirq_pending;
	#ifdef CONFIG_SMP
		unsigned int ipi_irqs[NR_IPI];
	#endif
	} ____cacheline_aligned irq_cpustat_t;

```

```c

	/*
	 * Contains default mappings for irq_cpustat_t, used by almost every
	 * architecture.  Some arch (like s390) have per cpu hardware pages and
	 * they define their own mappings for irq_stat.
	 *
	 * Keith Owens <kaos@ocs.com.au> July 2000.
	 */
	
	
	/*
	 * Simple wrappers reducing source bloat.  Define all irq_stat fields
	 * here, even ones that are arch dependent.  That way we get common
	 * definitions instead of differing sets for each arch.
	 */
	
	#ifndef __ARCH_IRQ_STAT
	extern irq_cpustat_t irq_stat[];		/* defined in asm/hardirq.h */
	#define __IRQ_STAT(cpu, member)	(irq_stat[cpu].member)
	#endif
	
	  /* arch independent irq_stat fields */
	#define local_softirq_pending() \
		__IRQ_STAT(smp_processor_id(), __softirq_pending)

```

So the `while()` loop is actually traversing `irq_stat[cpu].__softirq_pending` from bit 0, the variable is currently using bits 0-9, corresponding to 10 different types of softirq, each corresponding to a different softirq processing function. The following code in `linux-4.2/include/linux/interrupt.h` shows the definitions of SOFTIRQ.

```c

	/* PLEASE, avoid to allocate new softirqs, if you need not _really_ high
	   frequency threaded job scheduling. For almost all the purposes
	   tasklets are more than enough. F.e. all serial device BHs et
	   al. should be converted to tasklets, not to softirqs.
	 */
	
	enum
	{
		HI_SOFTIRQ=0,
		TIMER_SOFTIRQ,
		NET_TX_SOFTIRQ,
		NET_RX_SOFTIRQ,
		BLOCK_SOFTIRQ,
		BLOCK_IOPOLL_SOFTIRQ,
		TASKLET_SOFTIRQ,
		SCHED_SOFTIRQ,
		HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
				    numbering. Sigh! */
		RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */
	
		NR_SOFTIRQS
	};
	
	#define SOFTIRQ_STOP_IDLE_MASK (~(1 << RCU_SOFTIRQ))

```

For example, the corresponding **HI_SOFTIRQ** action is `tasklet_hi_action()`, and the corresponding **TASKLET_SOFTIRQ** action is exactly the same, which is what we usually say *tasklet*, as implemented in `linux-4.2/kernel/softirq.c`.

```c

	void __init softirq_init(void)
	{
		int cpu;
	
		for_each_possible_cpu(cpu) {
			per_cpu(tasklet_vec, cpu).tail =
				&per_cpu(tasklet_vec, cpu).head;
			per_cpu(tasklet_hi_vec, cpu).tail =
				&per_cpu(tasklet_hi_vec, cpu).head;
		}
	
		open_softirq(TASKLET_SOFTIRQ, tasklet_action);
		open_softirq(HI_SOFTIRQ, tasklet_hi_action);
	}

	void open_softirq(int nr, void (*action)(struct softirq_action *))
	{
		softirq_vec[nr].action = action;
	}

```

As we can see, `irq_stat[cpu].__softirq_pending` is a per-cpu type variable, because for the SMP system, each processor can independently handle its incoming external interrupts, also each of them has its own hardirq and softirq stack. 

## TASKLET_SOFTIRQ

We have always being saying hardirq stage identifies softirqs, what actually means identifying a softirq? In fact, it means calling `tasklet_schedule()` to set **TASKLET_SOFTIRQ** bit.

```c

	static inline void tasklet_schedule(struct tasklet_struct *t)
	{
		if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
			__tasklet_schedule(t);
	}

	void __tasklet_schedule(struct tasklet_struct *t)
	{
		unsigned long flags;
	
		local_irq_save(flags);
		t->next = NULL;
		*__this_cpu_read(tasklet_vec.tail) = t;
		__this_cpu_write(tasklet_vec.tail, &(t->next));
		raise_softirq_irqoff(TASKLET_SOFTIRQ);
		local_irq_restore(flags);
	}

	/*
	 * This function must run with irqs disabled!
	 */
	inline void raise_softirq_irqoff(unsigned int nr)
	{
		__raise_softirq_irqoff(nr);
	
		/*
		 * If we're in an interrupt or softirq, we're done
		 * (this also catches softirq-disabled code). We will
		 * actually run the softirq once we return from
		 * the irq or softirq.
		 *
		 * Otherwise we wake up ksoftirqd to make sure we
		 * schedule the softirq soon.
		 */
		if (!in_interrupt())
			wakeup_softirqd();
	}

	void __raise_softirq_irqoff(unsigned int nr)
	{
		trace_softirq_raise(nr);
		or_softirq_pending(1UL << nr);
	}

	/*
	 * we cannot loop indefinitely here to avoid userspace starvation,
	 * but we also don't want to introduce a worst case 1/HZ latency
	 * to the pending events, so lets the scheduler to balance
	 * the softirq load for us.
	 */
	static void wakeup_softirqd(void)
	{
		/* Interrupts are disabled: no need to stop preemption */
		struct task_struct *tsk = __this_cpu_read(ksoftirqd);
	
		if (tsk && tsk->state != TASK_RUNNING)
			wake_up_process(tsk);
	}

```

The `ksoftirqd` is spawned as a per-cpu thread, registered in `linux-4.2/kernel/softirq.c`.
 
```c

	static void run_ksoftirqd(unsigned int cpu)
	{
		local_irq_disable();
		if (local_softirq_pending()) {
			/*
			 * We can safely run softirq on inline stack, as we are not deep
			 * in the task stack here.
			 */
			__do_softirq();
			local_irq_enable();
			cond_resched_rcu_qs();
			return;
		}
		local_irq_enable();
	}

	static struct smp_hotplug_thread softirq_threads = {
		.store			= &ksoftirqd,
		.thread_should_run	= ksoftirqd_should_run,
		.thread_fn		= run_ksoftirqd,
		.thread_comm		= "ksoftirqd/%u",
	};
	
	static __init int spawn_ksoftirqd(void)
	{
		register_cpu_notifier(&cpu_nfb);
	
		BUG_ON(smpboot_register_percpu_thread(&softirq_threads));
	
		return 0;
	}

```

The following is a dump of the ksoftirqd kernel threads:

```console

	coryxie: ~ $ ps aux|grep ksoftirqd
	root         3  0.0  0.0      0     0 ?        S    Apr27   0:33 [ksoftirqd/0]
	root        10  0.0  0.0      0     0 ?        S    Apr27   0:15 [ksoftirqd/1]
	root        15  0.0  0.0      0     0 ?        S    Apr27   0:16 [ksoftirqd/2]
	root        19  0.0  0.0      0     0 ?        S    Apr27   0:15 [ksoftirqd/3]
	root        23  0.0  0.0      0     0 ?        S    Apr27   0:06 [ksoftirqd/4]
	root        27  0.0  0.0      0     0 ?        S    Apr27   0:05 [ksoftirqd/5]
	root        31  0.0  0.0      0     0 ?        S    Apr27   0:06 [ksoftirqd/6]
	root        35  0.0  0.0      0     0 ?        S    Apr27   0:06 [ksoftirqd/7]

```






