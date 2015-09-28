---
layout: post
title: "Linux Kernel CONFIG_LOCKDEP - Kernel Lock Validator"
description:
headline:
modified: 2015-09-28
category: KernelDevel
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

## CONFIG_LOCKDEP implementation description

As shown in FIG. 2, the kernel lock validator 130 can comprise a control module 205, a lock status module 210, a rules module 215 and a hash table 220. The control module 205 can be configured to manage and provide the functionality of the kernel lock validator 130. The control module 205, as with the other noted modules, can be implemented as a software routine (applet, program, etc.), a hardware component (ASIC, FPGA, etc.) or a combinations thereof.

<img src="{{ site.url }}/images/2015-09-28-1/FIG2.png" alt="the kernel lock validator">

The control module 205 can also be configured to track lock-types as opposed to tracking each individual instance of the lock-type. The lock-type is registered when it used for the first time after bootup and all subsequent uses of the registered lock-type will be attached to this lock-type. Accordingly, each lock type is assigned a specific key based on the registration of the lock-type. The control module 205 can be configured to create a static variable (e.g., (_key)) for statically declared locks and uses the address of the statically declared lock as the key. The control module 205 can maintain the information regarding the lock types and associated keys in the lock status module 210.

The control module 205 can be further configured to track lock-type usage by using a series of state bits: (1) ever held in hard interrupt context (hardirq-safe); (2) ever held in soft interrupt context (softirg-safe); (3) ever held in hard interrupt with interrupts enabled (hardirq-unsafe); (4) ever held with soft interrupts and hard interrupts enabled (softirq-unsafe); and (5) ever used (!unused). With these state bits, the control module 205 can be configured to test each locking operation with the predetermined rules stored in the rules module 215.

The rules module 215 can be configured to store the rules that the control module 205 uses to check each lock operation. The rules module 215 can be configured to stores rules such as a softirq-unsafe is automatically a hardirq-unsafe. Another rule is that following states are exclusive, i.e., can only be set for any lock type: (1) hardirq-safe and hardirq-unsafe; and (2) softirq-safe and softirq-unsafe. Yet another rule is that the same lock-type cannot be acquired twice as well as two locks cannot be acquired in different order. Yet another rule is that the following lock dependencies are not permitted: (1) hardirq-safe to hardirq-unsafe; and (2) softirq-safe to softirq-unsafe. Accordingly, the above rules are enforced for any locking sequence that occurs in the kernel 120: when acquiring a new lock, the control module 205 checks whether there is any rule violation between the new lock and any of the held locks.

There are additional rules for when a lock type changes state. One rule is if a new hardirq-safe lock is discovered, the lock validator determines whether it took any hardirq-unsafe lock in the past. A second change of state rule is if a new softirq-safe lock is discovered, the control module 205 determines whether it took any softirq-unsafe lock in the past. A third change of state rule is if a new hardirq-unsafe lock is discovered, the lock validator determines whether any hardirq-safe lock took it in the past. A fourth change of state rule is if a new softirq-unsafe lock is discovered, the lock validator determines whether any softirq-safe lock took it in the past. It should be readily that the list of rules for exemplary and new rules or existing rules modified without departing from the scope of the claimed invention.

As there are multiple rules, the ordering of the rules as they are tested is an additional requirement. Accordingly, the control module 205 can be configured to maintain two lists. A first list, a “before” list contains all locks which have ever been held when the lock of interest (referred to a L) is acquired. As a result, the before list contains the keys of all locks acquired before the L, the lock of interest. The second list, an “after” list, holds all locks acquired while the lock of interest is held. The before and after lists can be implemented using data structures such as a buffer, linked lists, and other similar referencing structures.

FIG. 3 depicts a block diagram of a first and second data structure 305, 310, respectively for implementing storing the before list and the after list. As shown in FIG. 3, first data structure 305 and second data structure 310 can be implemented as a series of memory spaces 315. The size of the memory space 315 can be dependent on the type of processor. More particularly, a 32-bit processor may indicate that the size of the memory space 315 is a 32-bit word as an example. In other embodiments, a user can alter the size of the memory space 315.

<img src="{{ site.url }}/images/2015-09-28-1/FIG3.png" alt="the before list and after list">

As shown in FIG. 4, the control module 205 can receive notification of an acquisition of a lock, L, by a process, thread, etc., in step 405. Alternatively, the control module 205 can also receive notification that a lock is changing state, e.g., being released.

<img src="{{ site.url }}/images/2015-09-28-1/FIG4.png" alt="the control module">

In step 410, the control module 205 can be configured to generate a 64-bit hash value of the sequence with the lock, L, as a temporary last entry into the data structure that holds the list of currently held locks. The hash value can be regarded as a ‘frontside cache’ so as to avoid testing of all held locks against the set of rules. The control module 205 can be configured to keep a ‘rolling’ 64-bit hash of the current ‘stack of locks’, which is updated when a lock in the stack is released or when a new lock is acquired. The update of the update 64-bit hash does not need to be recalculate the full has of all held locks because the hash generation step is reversible.

In step 415, the control module 205 can be configured to search the hash table 220 with the generated 64-bit hash value. If there is match, in step 420, the control module 205 knows that this sequence had previously been validated and the lock, L, is entered into the data structure that holds the list of currently held locks, in step 425. The control module 205 also updates the after-lists of each currently held lock with the acquisition of lock, L, in step 430.

Otherwise, if there is not a match, in step 420, the control module 205 can be configured to test the lock, Lt, against the rules stored in the rules module 220, in step 435. More particularly, the control module 205 can be configured to validate a new dependency against all existing dependencies. For example, if L2 has been taken after L1 in the past, and a L2=>L1 dependency is added, this condition is determined to be a violation. In essence, new dependencies are only added when they do not conflict with existing rules (or dependencies). Accordingly, the control module 205 can set the state of the lock, L, being acquired. Subsequently, the control module 205 can apply the rules stored in the rules module 215.

Alternatively, if the lock, L, is changing state, the control module 205 can test the lock, L, against the rules for lock changing state.

If the lock, L, did not pass the rules, in step 440, the acquiring entity is informed of the error and is disallowed from acquiring the lock, in step 445. Otherwise, in step 450, the control module 205 can be configured to create data structures for the before list 305 and after list 310. The control module 205 can also store the lock, L, in the data structure for storing currently held locks.

In step 450, the control module 205 can generate a 64-bit hash value on the current sequence of the list of currently held locks and subsequently stored in the hash table 220.


## Credits

Part of this blog is mostly an edit from [http://www.google.com/patents/US8145903](http://www.google.com/patents/US8145903 "Method and system for a kernel lock validator") for its authority.



