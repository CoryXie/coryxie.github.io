# CONFIG_LOCKDEP Kernel Lock Validator

Locks are a primitive utilized by the kernel to manage the resources of an operating system. Without locks, different parts of a system can collide when trying to access the same resources, leading to data corruption and general chaos. However, managing locks is a challenging programming task because the number of instances of locks, the type of locks and the cost of not resolving deadlocks.

The fundamental issue surrounding locking is the need to provide synchronization in certain code paths in the kernel. These code paths, called critical sections, require some combination of concurrency or reentrancy protection and proper ordering with respect to other events. The typical result without proper locking is called a race condition. As a simple example, consider two locks L1 and L2. Any code which requires both locks must take care to acquire the locks in the right order. If one function acquires L1 before L2, but another function acquires them in the opposite order, eventually the system will find itself in a situation where each function has acquired one lock and is blocked waiting for the other — a deadlock happens.

This is not to say that the only locking issues arise from SMP (symmetric multiprocessing). Interrupt handlers create locking issues and any code can block (go to sleep). Of these, only SMP is considered true concurrency, i.e., only with SMP can two things actually occur at the exact same time. The other situations — interrupt handlers, preempt-kernel and blocking methods — provide pseudo concurrency as code is not actually executed concurrently, but separate code can mangle one another's data.

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

The above rules require massive amounts of runtime checking. If this checking were done for every lock taken and for every interrupt (“irqs”)-enabled event, it would render the system practically unusabe. The complexity of checking is O(N2). Accordingly, tens of thousand of check would be performed for every event even with just a few hundred lock-types. The lock validator **CONFIG_LOCKDEP** resolves this issue by checking any given ‘locking scenario’ (unique sequence of locks taken after each other) only once. A simple stack of held locks is maintained, and a lightweight 64-bit hash value is calculated when the sequence (or chain) is validated for the first time. This hash value, which is unique for every lock chain. The hash value is then put into a hash table, which can be checked in a lock-free manner. If the locking chain occurs again later on, the hash table can notify the lock validator not to validate the chain again.

As there are multiple rules, the ordering of the rules as they are tested is an additional requirement. Accordingly, the control module of the lock validator can be configured to maintain two lists. 

>* A first list, a “before” list contains all locks which have ever been held when the lock of interest (referred to a L) is acquired. As a result, the before list contains the keys of all locks acquired before the L, the lock of interest. 
>* The second list, an “after” list, holds all locks acquired while the lock of interest is held. 

The before and after lists can be implemented using data structures such as a buffer, linked lists, and other similar referencing structures.