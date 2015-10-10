---
layout: post
title: "Linux Kernel CONFIG_LOCKDEP - Kernel Lock Validator"
description:
headline:
modified: 2015-09-28
category: KernelDev
tags: [Kernel]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

Locks are a primitive utilized by the kernel to manage the resources of an operating system. Without locks, different parts of a system can collide when trying to access the same resources, leading to data corruption and general chaos. However, managing locks is a challenging programming task because the number of instances of locks, the type of locks and the cost of not resolving deadlocks.

The fundamental issue surrounding locking is the need to provide synchronization in certain code paths in the kernel. These code paths, called critical sections, require some combination of concurrency or reentrancy protection and proper ordering with respect to other events. The typical result without proper locking is called a race condition. As a simple example, consider two locks L1 and L2. Any code which requires both locks must take care to acquire the locks in the right order. If one function acquires L1 before L2, but another function acquires them in the opposite order, eventually the system will find itself in a situation where each function has acquired one lock and is blocked waiting for the other — a deadlock happens.

This is not to say that the only locking issues arise from SMP (symmetric multiprocessing). Interrupt handlers create locking issues and any code can block (go to sleep). Of these, only SMP is considered true concurrency, i.e., only with SMP can two things actually occur at the exact same time. The other situations — interrupt handlers, preempt-kernel and blocking methods — provide pseudo concurrency as code is not actually executed concurrently, but separate code can mangle one another's data.

## CONFIG_LOCKDEP theory of operations

The Linux kernel provides a lock validator by way of **CONFIG_LOCKDEP** to prevent resource access conflicts in the kernel (the same mechanism can also be used in users-pace). More particularly, the lock validator can be configured to track the state of locks-types and it maintains dependencies between the different lock-types. The lock validator can also be configured to maintain a rolling proof that the state of the lock-types and the dependencies are correct.

The lock validator can be configured to track lock-types as opposed to tracking each individual instance of the lock-type. Unlike a lock instantiation, the lock-type is registered when it used for the first time after bootup and all subsequent uses of the registered lock-type will be attached to this lock-type. Accordingly, each lock type is assigned a specific key based on the registration of the lock-type. The lock validator can be configured to create a static variable (e.g., (_key)) for statically declared locks and uses the address of the statically declared lock as the key.

The lock validator can also be configured to track lock-type usage by using five separate state bits: 

>* (1) ever held in hard interrupt context (hardirq-safe); 
>* (2) ever held in soft interrupt context (softirg-safe); 
>* (3) ever held in hard interrupt with interrupts enabled (hardirq-unsafe); 
>* (4) ever held with soft interrupts and hard interrupts enabled (softirq-unsafe); and 
>* (5) ever used (!unused). 

With these state bits, the lock validator can be configured to test each locking operation. More particularly, the lock validator tests each locking operation against a number of rules. 

A softirq-unsafe is automatically a hardirq-unsafe is an example of one rule. 

Another rule is that following states are exclusive, i.e., can only be set for any lock type: 

>* (1) hardirq-safe and hardirq-unsafe; and 
>* (2) softirq-safe and softirq-unsafe. 

Yet another rule is that the same lock-type cannot be acquired twice as well as two locks cannot be acquired in different order. 

Yet another rule is that the following lock dependencies are not permitted: 

>* (1) hardirq-safe to hardirq-unsafe; and 
>* (2) softirq-safe to softirq-unsafe. 

Accordingly, the above rules are enforced for any locking sequence that occurs in the kernel: when acquiring a new lock, the lock validator checks whether there is any rule violation between the new lock and any of the held locks.

When a lock-type changes state, the following additional rules are tested: 

>* (1) if a new hardirq-safe lock is discovered, the lock validator determines whether it took any hardirq-unsafe lock in the past; 
>* (2) if a new softirq-safe lock is discovered, the lock validator determines whether it took any softirq-unsafe lock in the past; 
>* (3) if a new hardirq-unsafe lock is discovered, the lock validator determines whether any hardirq-safe lock took it in the past; and 
>* (4) if a new softirq-unsafe lock is discovered, the lock validator determines whether any softirq-safe lock took it in the past. 

Accordingly, the lock validator can be configured to report the rule violation where it can be used to modify the behavior of the software (e.g., kernel or application) to mitigate the effect of the lock violation.

The above rules require massive amounts of runtime checking. If this checking were done for every lock taken and for every interrupt ("irqs")-enabled event, it would render the system practically unusabe. The complexity of checking is O(N2). Accordingly, tens of thousand of check would be performed for every event even with just a few hundred lock-types. The lock validator **CONFIG_LOCKDEP** resolves this issue by checking any given ‘locking scenario’ (unique sequence of locks taken after each other) only once. A simple stack of held locks is maintained, and a lightweight 64-bit hash value is calculated when the sequence (or chain) is validated for the first time. This hash value, which is unique for every lock chain. The hash value is then put into a hash table, which can be checked in a lock-free manner. If the locking chain occurs again later on, the hash table can notify the lock validator not to validate the chain again.

As there are multiple rules, the ordering of the rules as they are tested is an additional requirement. Accordingly, the control module of the lock validator can be configured to maintain two lists. 

>* A first list, a "before" list contains all locks which have ever been held when the lock of interest (referred to a L) is acquired. As a result, the before list contains the keys of all locks acquired before the L, the lock of interest. 
>* The second list, an "after" list, holds all locks acquired while the lock of interest is held.

The before and after lists can be implemented using data structures such as a buffer, linked lists, and other similar referencing structures.

## CONFIG_LOCKDEP implementation design

As shown in FIG. 2, the kernel lock validator 130 can comprise a control module 205, a lock status module 210, a rules module 215 and a hash table 220. The control module 205 can be configured to manage and provide the functionality of the kernel lock validator 130. The control module 205, as with the other noted modules, can be implemented as a software routine (applet, program, etc.), a hardware component (ASIC, FPGA, etc.) or a combinations thereof.

<img src="{{ site.url }}/images/2015-09-28-1/US8145903-FIG2.png" alt="the kernel lock validator">

The control module 205 can also be configured to track lock-types as opposed to tracking each individual instance of the lock-type. The lock-type is registered when it used for the first time after bootup and all subsequent uses of the registered lock-type will be attached to this lock-type. Accordingly, each lock type is assigned a specific key based on the registration of the lock-type. The control module 205 can be configured to create a static variable (e.g., (_key)) for statically declared locks and uses the address of the statically declared lock as the key. The control module 205 can maintain the information regarding the lock types and associated keys in the lock status module 210.

The control module 205 can be further configured to track lock-type usage by using a series of state bits: (1) ever held in hard interrupt context (hardirq-safe); (2) ever held in soft interrupt context (softirg-safe); (3) ever held in hard interrupt with interrupts enabled (hardirq-unsafe); (4) ever held with soft interrupts and hard interrupts enabled (softirq-unsafe); and (5) ever used (!unused). With these state bits, the control module 205 can be configured to test each locking operation with the predetermined rules stored in the rules module 215.

The rules module 215 can be configured to store the rules that the control module 205 uses to check each lock operation. The rules module 215 can be configured to stores rules such as a softirq-unsafe is automatically a hardirq-unsafe. Another rule is that following states are exclusive, i.e., can only be set for any lock type: (1) hardirq-safe and hardirq-unsafe; and (2) softirq-safe and softirq-unsafe. Yet another rule is that the same lock-type cannot be acquired twice as well as two locks cannot be acquired in different order. Yet another rule is that the following lock dependencies are not permitted: (1) hardirq-safe to hardirq-unsafe; and (2) softirq-safe to softirq-unsafe. Accordingly, the above rules are enforced for any locking sequence that occurs in the kernel 120: when acquiring a new lock, the control module 205 checks whether there is any rule violation between the new lock and any of the held locks.

There are additional rules for when a lock type changes state. One rule is if a new hardirq-safe lock is discovered, the lock validator determines whether it took any hardirq-unsafe lock in the past. A second change of state rule is if a new softirq-safe lock is discovered, the control module 205 determines whether it took any softirq-unsafe lock in the past. A third change of state rule is if a new hardirq-unsafe lock is discovered, the lock validator determines whether any hardirq-safe lock took it in the past. A fourth change of state rule is if a new softirq-unsafe lock is discovered, the lock validator determines whether any softirq-safe lock took it in the past. It should be readily that the list of rules for exemplary and new rules or existing rules modified without departing from the scope of the claimed invention.

As there are multiple rules, the ordering of the rules as they are tested is an additional requirement. Accordingly, the control module 205 can be configured to maintain two lists. A first list, a “before” list contains all locks which have ever been held when the lock of interest (referred to a L) is acquired. As a result, the before list contains the keys of all locks acquired before the L, the lock of interest. The second list, an “after” list, holds all locks acquired while the lock of interest is held. The before and after lists can be implemented using data structures such as a buffer, linked lists, and other similar referencing structures.

FIG. 3 depicts a block diagram of a first and second data structure 305, 310, respectively for implementing storing the before list and the after list. As shown in FIG. 3, first data structure 305 and second data structure 310 can be implemented as a series of memory spaces 315. The size of the memory space 315 can be dependent on the type of processor. More particularly, a 32-bit processor may indicate that the size of the memory space 315 is a 32-bit word as an example. In other embodiments, a user can alter the size of the memory space 315.

<img src="{{ site.url }}/images/2015-09-28-1/US8145903-FIG3.png" alt="the before list and after list">

As shown in FIG. 4, the control module 205 can receive notification of an acquisition of a lock, L, by a process, thread, etc., in step 405. Alternatively, the control module 205 can also receive notification that a lock is changing state, e.g., being released.

<img src="{{ site.url }}/images/2015-09-28-1/US8145903-FIG4.png" alt="the control module">

In step 410, the control module 205 can be configured to generate a 64-bit hash value of the sequence with the lock, L, as a temporary last entry into the data structure that holds the list of currently held locks. The hash value can be regarded as a ‘frontside cache’ so as to avoid testing of all held locks against the set of rules. The control module 205 can be configured to keep a ‘rolling’ 64-bit hash of the current ‘stack of locks’, which is updated when a lock in the stack is released or when a new lock is acquired. The update of the update 64-bit hash does not need to be recalculate the full has of all held locks because the hash generation step is reversible.

In step 415, the control module 205 can be configured to search the hash table 220 with the generated 64-bit hash value. If there is match, in step 420, the control module 205 knows that this sequence had previously been validated and the lock, L, is entered into the data structure that holds the list of currently held locks, in step 425. The control module 205 also updates the after-lists of each currently held lock with the acquisition of lock, L, in step 430.

Otherwise, if there is not a match, in step 420, the control module 205 can be configured to test the lock, Lt, against the rules stored in the rules module 220, in step 435. More particularly, the control module 205 can be configured to validate a new dependency against all existing dependencies. For example, if L2 has been taken after L1 in the past, and a L2=>L1 dependency is added, this condition is determined to be a violation. In essence, new dependencies are only added when they do not conflict with existing rules (or dependencies). Accordingly, the control module 205 can set the state of the lock, L, being acquired. Subsequently, the control module 205 can apply the rules stored in the rules module 215.

Alternatively, if the lock, L, is changing state, the control module 205 can test the lock, L, against the rules for lock changing state.

If the lock, L, did not pass the rules, in step 440, the acquiring entity is informed of the error and is disallowed from acquiring the lock, in step 445. Otherwise, in step 450, the control module 205 can be configured to create data structures for the before list 305 and after list 310. The control module 205 can also store the lock, L, in the data structure for storing currently held locks.

In step 450, the control module 205 can generate a 64-bit hash value on the current sequence of the list of currently held locks and subsequently stored in the hash table 220.

## CONFIG_LOCKDEP implementation details

The `CONFIG_LOCKDEP` configuration is defined in `lib/Kconfig.debug` along with `CONFIG_DEBUG_LOCK_ALLOC` and `CONFIG_PROVE_LOCKING`.

```c

	config DEBUG_LOCK_ALLOC
		bool "Lock debugging: detect incorrect freeing of live locks"
		depends on DEBUG_KERNEL && TRACE_IRQFLAGS_SUPPORT && STACKTRACE_SUPPORT && LOCKDEP_SUPPORT
		select DEBUG_SPINLOCK
		select DEBUG_MUTEXES
		select LOCKDEP
		help
		 This feature will check whether any held lock (spinlock, rwlock,
		 mutex or rwsem) is incorrectly freed by the kernel, via any of the
		 memory-freeing routines (kfree(), kmem_cache_free(), free_pages(),
		 vfree(), etc.), whether a live lock is incorrectly reinitialized via
		 spin_lock_init()/mutex_init()/etc., or whether there is any lock
		 held during task exit.
	
	config PROVE_LOCKING
		bool "Lock debugging: prove locking correctness"
		depends on DEBUG_KERNEL && TRACE_IRQFLAGS_SUPPORT && STACKTRACE_SUPPORT && LOCKDEP_SUPPORT
		select LOCKDEP
		select DEBUG_SPINLOCK
		select DEBUG_MUTEXES
		select DEBUG_LOCK_ALLOC
		select TRACE_IRQFLAGS
		default n
		help
		 This feature enables the kernel to prove that all locking
		 that occurs in the kernel runtime is mathematically
		 correct: that under no circumstance could an arbitrary (and
		 not yet triggered) combination of observed locking
		 sequences (on an arbitrary number of CPUs, running an
		 arbitrary number of tasks and interrupt contexts) cause a
		 deadlock.
	
		 In short, this feature enables the kernel to report locking
		 related deadlocks before they actually occur.
	
		 The proof does not depend on how hard and complex a
		 deadlock scenario would be to trigger: how many
		 participant CPUs, tasks and irq-contexts would be needed
		 for it to trigger. The proof also does not depend on
		 timing: if a race and a resulting deadlock is possible
		 theoretically (no matter how unlikely the race scenario
		 is), it will be proven so and will immediately be
		 reported by the kernel (once the event is observed that
		 makes the deadlock theoretically possible).
	
		 If a deadlock is impossible (i.e. the locking rules, as
		 observed by the kernel, are mathematically correct), the
		 kernel reports nothing.
	
		 NOTE: this feature can also be enabled for rwlocks, mutexes
		 and rwsems - in which case all dependencies between these
		 different locking variants are observed and mapped too, and
		 the proof of observed correctness is also maintained for an
		 arbitrary combination of these separate locking variants.
	
		 For more details, see Documentation/locking/lockdep-design.txt.
	
	config LOCKDEP
		bool
		depends on DEBUG_KERNEL && TRACE_IRQFLAGS_SUPPORT && STACKTRACE_SUPPORT && LOCKDEP_SUPPORT
		select STACKTRACE
		select FRAME_POINTER if !MIPS && !PPC && !ARM_UNWIND && !S390 && !MICROBLAZE && !ARC && !SCORE
		select KALLSYMS
		select KALLSYMS_ALL

```

This means that once `CONFIG_DEBUG_LOCK_ALLOC` is selected, `CONFIG_LOCKDEP` configuration will also be selected. The following subsections will try to describe the implementation details for `CONFIG_LOCKDEP`.

### The glue structure - struct lockdep_map ###

The `struct lockdep_map` is the glue structure to link any kind of lockdep enabled locking mechanism with the `struct lock_class` structure wihch will be described in next subsection. Each such locking structure would have a `struct lockdep_map` embedded into its structure definition. For example, the `struct raw_spinlock` as defined in `<include/linux/spinlock_types.h>` takes a `struct lockdep_map dep_map` field as below.

```c

	typedef struct raw_spinlock {
		arch_spinlock_t raw_lock;
	#ifdef CONFIG_GENERIC_LOCKBREAK
		unsigned int break_lock;
	#endif
	#ifdef CONFIG_DEBUG_SPINLOCK
		unsigned int magic, owner_cpu;
		void *owner;
	#endif
	#ifdef CONFIG_DEBUG_LOCK_ALLOC
		struct lockdep_map dep_map;
	#endif
	} raw_spinlock_t;

```

Note that the `CONFIG_DEBUG_LOCK_ALLOC` is used to wrap the inclusion of `struct lockdep_map dep_map`. The following code snippet is extracted from `<include/linux/lockdep.h>`.

```c

	#define MAX_LOCKDEP_SUBCLASSES		8UL
	
	/*
	 * NR_LOCKDEP_CACHING_CLASSES ... Number of classes
	 * cached in the instance of lockdep_map
	 *
	 * Currently main class (subclass == 0) and signle depth subclass
	 * are cached in lockdep_map. This optimization is mainly targeting
	 * on rq->lock. double_rq_lock() acquires this highly competitive with
	 * single depth.
	 */
	#define NR_LOCKDEP_CACHING_CLASSES	2

	/*
	 * Lock-classes are keyed via unique addresses, by embedding the
	 * lockclass-key into the kernel (or module) .data section. (For
	 * static locks we use the lock address itself as the key.)
	 */
	struct lockdep_subclass_key {
		char __one_byte;
	} __attribute__ ((__packed__));
	
	struct lock_class_key {
		struct lockdep_subclass_key	subkeys[MAX_LOCKDEP_SUBCLASSES];
	};

	/*
	 * Map the lock object (the lock instance) to the lock-class object.
	 * This is embedded into specific lock instances:
	 */
	struct lockdep_map {
		struct lock_class_key		*key;
		struct lock_class		*class_cache[NR_LOCKDEP_CACHING_CLASSES];
		const char			*name;
	#ifdef CONFIG_LOCK_STAT
		int				cpu;
		unsigned long			ip;
	#endif
	};

```

Every lock in the system (including rwlocks and mutexes) is assigned a specific key. For locks which are declared statically, the address of the lock is used as the key. This is not initially explict for spinlocks as the static initializer for spinlocks look like the following:

```c

	#define SPINLOCK_MAGIC		0xdead4ead
	
	#define SPINLOCK_OWNER_INIT	((void *)-1L)
	
	#ifdef CONFIG_DEBUG_LOCK_ALLOC
	# define SPIN_DEP_MAP_INIT(lockname)	.dep_map = { .name = #lockname }
	#else
	# define SPIN_DEP_MAP_INIT(lockname)
	#endif
	
	#ifdef CONFIG_DEBUG_SPINLOCK
	# define SPIN_DEBUG_INIT(lockname)		\
		.magic = SPINLOCK_MAGIC,		\
		.owner_cpu = -1,			\
		.owner = SPINLOCK_OWNER_INIT,
	#else
	# define SPIN_DEBUG_INIT(lockname)
	#endif
	
	#define __RAW_SPIN_LOCK_INITIALIZER(lockname)	\
		{					\
		.raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,	\
		SPIN_DEBUG_INIT(lockname)		\
		SPIN_DEP_MAP_INIT(lockname) }
	
	#define __RAW_SPIN_LOCK_UNLOCKED(lockname)	\
		(raw_spinlock_t) __RAW_SPIN_LOCK_INITIALIZER(lockname)
	
	#define DEFINE_RAW_SPINLOCK(x)	raw_spinlock_t x = __RAW_SPIN_LOCK_UNLOCKED(x)

```

From the above code we can see that the `SPIN_DEP_MAP_INIT(lockname)` is only setting the `dep_map.name` field to the name of the lock. The `dep_map.key` will only be initialized to be the address of the lock when the corresponding lock class is registered, for which we will see the detailed code sequence later, but here we can just have a small snippet of the function `look_up_lock_class()` as defined in `kernel/locking/lockdep.c`.

```c

	/*
	 * Register a lock's class in the hash-table, if the class is not present
	 * yet. Otherwise we look it up. We cache the result in the lock object
	 * itself, so actual lookup of the hash should be once per lock object.
	 */
	static inline struct lock_class *
	look_up_lock_class(struct lockdep_map *lock, unsigned int subclass) {
	...
		/*
		 * Static locks do not have their class-keys yet - for them the key
		 * is the lock object itself:
		 */
		if (unlikely(!lock->key))
			lock->key = (void *)lock;
	...
	}

```

Locks which are allocated dynamically (as most locks embedded within structures are) cannot be tracked that way, however; there may be vast numbers of addresses involved, and, in any case, all locks associated with a specific structure field should be mapped to a single key. This is done by recognizing that these locks are initialized at run time, so, for example, raw_spin_lock_init() is redefined as:

```c

	#ifdef CONFIG_DEBUG_SPINLOCK
	  extern void __raw_spin_lock_init(raw_spinlock_t *lock, const char *name,
					   struct lock_class_key *key);
	# define raw_spin_lock_init(lock)				\
	do {								\
		static struct lock_class_key __key;			\
									\
		__raw_spin_lock_init((lock), #lock, &__key);		\
	} while (0)
	
	#else
	# define raw_spin_lock_init(lock)				\
		do { *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock); } while (0)
	#endif

```

Thus, for each lock initialization, this code creates a static variable (__key) and uses its address as the key identifying the type of the lock. Since any particular type of lock tends to be initialized in a single place, this trick associates the same key with every lock of the same type. The `dep_map` will be further initialized in `__raw_spin_lock_init()` as below:

```c

	void __raw_spin_lock_init(raw_spinlock_t *lock, const char *name,
				  struct lock_class_key *key)
	{
	#ifdef CONFIG_DEBUG_LOCK_ALLOC
		/*
		 * Make sure we are not reinitializing a held lock:
		 */
		debug_check_no_locks_freed((void *)lock, sizeof(*lock));
		lockdep_init_map(&lock->dep_map, name, key, 0);
	#endif
		lock->raw_lock = (arch_spinlock_t)__ARCH_SPIN_LOCK_UNLOCKED;
		lock->magic = SPINLOCK_MAGIC;
		lock->owner = SPINLOCK_OWNER_INIT;
		lock->owner_cpu = -1;
	}

```

### The lock type object - struct lock_class ###

A lock-type is defined as a `struct lock_class` structure in Linux Kernel, which is the basic object the validator operates upon.

```c

	/*
	 * The lock-class itself:
	 */
	struct lock_class {
		/*
		 * class-hash:
		 */
		struct list_head		hash_entry;
	
		/*
		 * global list of all lock-classes:
		 */
		struct list_head		lock_entry;
	
		struct lockdep_subclass_key	*key;
		unsigned int			subclass;
		unsigned int			dep_gen_id;
	
		/*
		 * IRQ/softirq usage tracking bits:
		 */
		unsigned long			usage_mask;
		struct stack_trace		usage_traces[XXX_LOCK_USAGE_STATES];
	
		/*
		 * These fields represent a directed graph of lock dependencies,
		 * to every node we attach a list of "forward" and a list of
		 * "backward" graph nodes.
		 */
		struct list_head		locks_after, locks_before;
	
		/*
		 * Generation counter, when doing certain classes of graph walking,
		 * to ensure that we check one node only once:
		 */
		unsigned int			version;
	
		/*
		 * Statistics counter:
		 */
		unsigned long			ops;
	
		const char			*name;
		int				name_version;
	
	#ifdef CONFIG_LOCK_STAT
		unsigned long			contention_point[LOCKSTAT_POINTS];
		unsigned long			contending_point[LOCKSTAT_POINTS];
	#endif
	};

```

A class of locks is a group of locks that are logically the same with respect to locking rules, even if the locks may have multiple (possibly tens of thousands of) instantiations. For example a lock in the inode struct is one class, while each inode has its own instantiation of that lock class.

The validator tracks the 'state' of lock-classes, and it tracks dependencies between different lock-classes. The validator maintains a rolling proof that the state and the dependencies are correct.

#### `lock_class::usage_mask` ####

The 'state' is is tracked by `usage_mask` as defined below:

```c

	/*
	 * Lock-class usage-state bits:
	 */
	enum lock_usage_bit {
	#define LOCKDEP_STATE(__STATE)		\
		LOCK_USED_IN_##__STATE,		\
		LOCK_USED_IN_##__STATE##_READ,	\
		LOCK_ENABLED_##__STATE,		\
		LOCK_ENABLED_##__STATE##_READ,
	#include "lockdep_states.h"
	#undef LOCKDEP_STATE
		LOCK_USED,
		LOCK_USAGE_STATES
	};

```

The `lockdep_states.h` defines the lock states:

- LOCKDEP_STATE(HARDIRQ)
- LOCKDEP_STATE(SOFTIRQ)
- LOCKDEP_STATE(RECLAIM_FS)

So we are actually having `LOCK_USED_IN_HARDIRQ`, `LOCK_USED_IN_HARDIRQ_READ`, `LOCK_ENABLED_HARDIRQ`, `LOCK_ENABLED_HARDIRQ_READ` etc, as well as the `LOCK_USED`.

The following code shows how the states are printed when reporting locking rule violations:

```c

	static inline unsigned long lock_flag(enum lock_usage_bit bit)
	{
		return 1UL << bit;
	}
	
	static char get_usage_char(struct lock_class *class, enum lock_usage_bit bit)
	{
		char c = '.';
	
		if (class->usage_mask & lock_flag(bit + 2))
			c = '+';
		if (class->usage_mask & lock_flag(bit)) {
			c = '-';
			if (class->usage_mask & lock_flag(bit + 2))
				c = '?';
		}
	
		return c;
	}

```

The bit position indicates STATE, STATE-read, for each of the states listed above, and the character displayed in each indicates:

  - '.'  acquired while irqs disabled and not in irq context
  - '-'  acquired in irq context
  - '+'  acquired with irqs enabled
  - '?'  acquired in irq context with irqs enabled.

#### `lock_class::locks_after` and `lock_class::locks_before` ####


Unlike an lock instantiation, the lock-class itself never goes away: when a lock-class is used for the first time after bootup it gets registered, and all subsequent uses of that lock-class will be attached to this lock-class.

#### The lock() and unlock() operations ####

Take the `__raw_spin_lock()` as an example, when `CONFIG_DEBUG_LOCK_ALLOC` is enabled, after disabling preemption, it calls a generic `spin_acquire()` with the `dep_map` as its 1st parameter.

```c

	static inline void __raw_spin_lock(raw_spinlock_t *lock)
	{
		preempt_disable();
		spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
		LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
	}

```

In the `spin_acquire()` function, the validator does a lot of things, the most notable one is that the code intercepts the locking operation and performs a number of tests:

- The code looks at all other locks which are already held when a new lock is taken. For all of those locks, the validator looks for a past occurrence where any of them were taken after the new lock. If any such are found, it indicates a violation of locking order rules, and an eventual deadlock.

- A stack of currently-held locks is maintained, so any lock being released should be at the top of the stack; anything else means that something strange is going on.

- Any spinlock which is acquired by a hardware interrupt handler can never be held when interrupts are enabled. Consider what happens when this rule is broken. A kernel function, running in process context, acquires a specific lock. An interrupt arrives, and the associated interrupt handler runs on the same CPU; that handler then attempts to acquire the same lock. Since the lock is unavailable, the handler will spin, waiting for the lock to become free. But the handler has preempted the only code which will ever free that lock, so it will spin forever, deadlocking that processor.

- To catch problems of this type, the validator records two bits of information for every lock it knows about: (1) whether the lock has ever been acquired in hardware interrupt context, and (2) whether the lock is ever held by code which runs with hardware interrupts enabled. If both bits are set, the lock is being used erroneously and an error is signaled.

Similar tests are made for software interrupts, which present the same problems.

There are mappings for different types of locks to the generic `lock_acquire()` function as defined in `<include/linux/lockdep.h>`, for example, the spinlock operations are mapped like below:

```c

	/*
	 * Map the dependency ops to NOP or to real lockdep ops, depending
	 * on the per lock-class debug mode:
	 */
	
	#define lock_acquire_exclusive(l, s, t, n, i)		lock_acquire(l, s, t, 0, 1, n, i)
	#define lock_acquire_shared(l, s, t, n, i)		lock_acquire(l, s, t, 1, 1, n, i)
	#define lock_acquire_shared_recursive(l, s, t, n, i)	lock_acquire(l, s, t, 2, 1, n, i)
	
	#define spin_acquire(l, s, t, i)		lock_acquire_exclusive(l, s, t, NULL, i)
	#define spin_acquire_nest(l, s, t, n, i)	lock_acquire_exclusive(l, s, t, n, i)
	#define spin_release(l, n, i)			lock_release(l, n, i)

```

The generic `lock_acquire()` and similar `lock_release()` functions are defined in `kernel/locking/lockdep.c`:

```c

	/*
	 * We are not always called with irqs disabled - do that here,
	 * and also avoid lockdep recursion:
	 */
	void lock_acquire(struct lockdep_map *lock, unsigned int subclass,
				  int trylock, int read, int check,
				  struct lockdep_map *nest_lock, unsigned long ip)
	{
		unsigned long flags;
	
		if (unlikely(current->lockdep_recursion))
			return;
	
		raw_local_irq_save(flags);
		check_flags(flags);
	
		current->lockdep_recursion = 1;
		trace_lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip);
		__lock_acquire(lock, subclass, trylock, read, check,
			       irqs_disabled_flags(flags), nest_lock, ip, 0);
		current->lockdep_recursion = 0;
		raw_local_irq_restore(flags);
	}

	void lock_release(struct lockdep_map *lock, int nested,
				  unsigned long ip)
	{
		unsigned long flags;
	
		if (unlikely(current->lockdep_recursion))
			return;
	
		raw_local_irq_save(flags);
		check_flags(flags);
		current->lockdep_recursion = 1;
		trace_lock_release(lock, ip);
		if (__lock_release(lock, nested, ip))
			check_chain_key(current);
		current->lockdep_recursion = 0;
		raw_local_irq_restore(flags);
	}

```

## Credits

The 1st part of this blog is mostly an edit from [http://www.google.com/patents/US8145903](http://www.google.com/patents/US8145903 "Method and system for a kernel lock validator") for its authority. The 2nd part of this blog is code reading notes by me but with some edits from [https://lwn.net/Articles/185666/](https://lwn.net/Articles/185666/ "The kernel lock validator").



