
## Linux Device Model (LDM)

* Explain about the Linux Device Model (LDM)?

The initial motivation for the device model was this final point: **providing an accurate device tree to facilitate power management**.

A *unified device model* was added in Linux kernel to provide a single mechanism for representing devices and describing their topology in the system. Such a system provides several benefits:

>- Minimization of code duplication
>- A mechanism for providing common facilities, such as reference counting
>- Capability to determine all the devices in the system, view their status and power state, see to what bus they are attached, and which driver is responsible for them
>- The capability to generate a complete and valid tree of the entire device structure of the system, including all buses and interconnections
>- The capability to link devices to their drivers and vice versa
>- Categorize devices by their kind ("classes"), such as input device, without the need to understand the physical device topology
>- Power Management - The capability to walk the tree of devices from the leaves up to the root, powering down devices in the correct order

The device model brings with it a whole new vocabulary to describe its data structures. A quick overview of some device model terms appears below.


>- **device**

A physical or virtual object which attaches to a (possibly virtual) bus.

>- **driver**

A software entity which may probe for and be bound to devices, and which can perform certain management functions.

>- **bus**

A device which serves as an attachment point for other devices.

>- **class**

A particular type of device which can be expected to perform in certain ways. Classes might include disks, partitions, serial ports, etc.

>- **subsystem**

A top-level view of the system's structure. Subsystems used in the kernel include devices (a hierarchical view of all devices on the system), bus (a bus-oriented view), class(devices by class), net (the networking subsystem), and others. The best way to think of a subsystem, perhaps, is as a particular view into the device model data structure rather than a physical component of the system. The same objects (devices, usually) show up in most subsystems, but they are organized differently.

References:

>- [Linux Device Model](http://linuxburps.blogspot.com/2013/12/linux-device-model.html) 

* Explain about about ksets, kobjects and ktypes. How are they related?

The fundamental task of the device model is to maintain a set of internal data structures which reflect the architecture and state of the underlying system. The device model works by tracking system configuration changes (hardware and software) and maintaining a complex "web woven by a spider on drugs" data structure to represent it all.

In order to understand device model, it is of utmost importance to first understand the low lying structures and their relationship with each other. 

>- **Kobjects**

At the heart of the device model is the `kobject`, short for kernel object, which is represented by `struct kobject` and defined in `<linux/kobject.h>`. It provides basic facilities, such as reference counting, a name, and a parent pointer, enabling the creation of a hierarchy of objects. 

Even though, in most of cases, you will never have to manipulate a `kobject` directly, it is hard to dig very deeply into the driver model without encountering them. Kobjects are usually embedded in other structures. Some of the important fields are:

```c

	struct kobject
	|-- name (string)
	|-- parent (kobject’s parent)
	|-- ktype (type associated with a kobject)
	|-- kset (group of kobjects all of which are embedded in structures of the same type)
	|-- sd (points to a sysfs_dirent structure that represents this kobject in sysfs.)
	|-- kref (provides reference counting)

```

It is the glue that holds much of the device model and its sysfs interface together.

For initialization and setup of kobjects, following functions exist:

        void kobject_init(struct kobject *kobj);

Kobject users must, at a minimum, set the name of the kobject; this is the name that will be used in sysfs entries.
        
		kobject_set_name(struct kobject *kobj, "The name");

Following functions manage the reference counts of kobjects:

        struct kobject *kobject_get(struct kobject *kobj);
        void kobject_put(struct kobject *kobj);

To create sysfs entries:

    int kobject_add(struct kobject *kobj);
         void kobject_del(struct kobject *kobj);

There is a `kobject_register()` function, which is really just the combination of the calls to `kobject_init()` and `kobject_add()`. Similarly, `kobject_unregister()` will call `kobject_del()`, then call `kobject_put()` to release the initial reference created with `kobject_register()` (or really `kobject_init()`).

>- **Ktypes**

Kobjects are associated with a specific type, called a `ktype`, short for kernel object type. Ktypes are represented by `struct kobj_type` and defined in `<linux/kobject.h>`.

```c

	struct kobj_type
	|-- release (pointer points to the deconstructor)
	|-- sysfs_ops (describes the behavior of sysfs files on read and write)
	|-- default_attrs (default attributes associated with this kobject)

```

Ktypes have the simple job of describing default behavior for a family of kobjects.

Instead of each `kobject` defining its own behavior, the behavior is stored in a `ktype`, and kobjects of the same "type" point at the same ktype structure, thus sharing the same behavior.

Every `kobject` needs to have an associated `kobj_type` structure and every `kobject` must have a `release()` method, and the `kobject` must persist (in a consistent state) until that method is called. If these constraints are not met, the code is flawed.

>- **Ksets**

Ksets, short for kernel object sets, are aggregate collections of kobjects. Ksets work as the base container class for a set of kernel objects, collecting related kobjects, such as "all block devices", together in a single place. 

The `kset` pointer in `struct kobject` points at a `kobject`'s associated `kset`. ksets are represented by the `kset` structure, which is declared in `<linux/kobject.h>`.

```c

	struct kset
	|-- list (linked list of all kobjects in this kset)
	|-- kobj (kobject representing the base class for this set)
	|-- uevent_ops (describes the hotplug behavior of kobjects in this kset)

```

Ksets group related kernel objects together, whereas ktypes enable kernel objects (functionally related or not) to share common operations.

The distinction is kept to allow kobjects of identical ktypes to be grouped into different ksets.

A `kset` serves these functions:

1. It serves as a bag containing a group of identical objects. A `kset` can be used by the kernel to track "all block devices" or "all PCI device drivers."

2. A `kset` is the directory-level glue that holds the device model (and `sysfs`) together. Every `kset` contains a `kobject` which can be set up to be the parent of other kobjects; in this way the device model hierarchy is constructed.

3. Ksets can support the "hotplugging" of kobjects and influence how hotplug events are reported to user space.

For initialization and setup of ksets, following functions exist:

```c

    void kset_init(struct kset *kset);
    int kset_add(struct kset *kset);
    int kset_register(struct kset *kset);
    void kset_unregister(struct kset *kset);

```

Following functions manage the reference counting of ksets :

```c

    struct kset *kset_get(struct kset *kset);
    void kset_put(struct kset *kset);

```

A `kset`, too, has a name, which is stored in the embedded `kobject` whose name is set by:

    kobject_set_name(my_set->kobj, "The name");

* Questions about `sysfs`.

## Linux Boot Sequence

* Explain about the Linux boot sequence in case of ARM architecture?
* How are the command line arguments passed to Linux kernel by the u-boot (bootloader)?
* Explain about ATAGS?
* Explain about command line arguments that are passed to linux kernel and how/where they are parsed in kernel code?
* Explain about device tree.

## Interrupts in Linux

* Explain about the interrupt mechanism in linux?
* What are the APIs that are used to register an interrupt handler?
* How do you register an interrupt handler on a shared IRQ line?
* Explain about the flags that are passed to `request_irq()`.
* Explain about the internals of Interrupt handling in case of Linux running on ARM.
* What are the precautions to be taken while writing an interrupt handler?
* Explain interrupt sequence in detail starting from ARM to registered interrupt handler.
* What is bottom half and top half.
* What is request_threaded_irq()
* If same interrupts occurs in two cpu how are they handled?
* How to synchronize data between 'two interrupts' and 'interrupts and process'.
* How are nested interrupts handled?
* How is task context saved during interrupt.

## Bottom-half Mechanisms in Linux
* What are the different bottom-half mechanisms in Linux?
* Softirq, Tasklet and Workqueues
* What are the differences between Softirq/Tasklet and Workqueue? Given an example what you prefer to use?
* What are the differences between softirqs and tasklets?
* Softirq is guaranteed to run on the CPU it was scheduled on, where as tasklets don’t have that guarantee. 
* The same tasklet can't run on two separate CPUs at the same time, where as a softirq can. 
* When are these bottom halfs executed?
* Explain about the internal implementation of softirqs?
* Bottom-halves in Linux - Part 1: Softirqs
* Explain about the internal implementation of tasklets?
* Bottom-halves in Linux - Part 2: Tasklets
* Explain about the internal implementation of workqueues?
* Bottom-halves in Linux - Part 3: Workqueues
* Explain about the concurrent work queues.

## Linux Memory Management 

* What are the differences between vmalloc and kmalloc? Which is preferred to use in device drivers?
* What are the differences between slab allocator and slub allocator?
* What is boot memory allocator?
* How do you reserve block of memory?
* What is virtual memory and what are the advanatages of using virtual memory?
* What's paging and swapping?
* Is it better to enable swapping in embedded systems? and why?
* What is the page size in Linux kernel in case of 32-bit ARM architecture?
* What is page frame?
* What are the different memory zones and why does different zones exist?
* What is high memory and when is it needed?
* Why is high memory zone not needed in case of 64-bit machine?
* How to allocate a page frame from high memory?
* In ARM, an abort exception if generated, if the page table doesn't contain a virtual to physical map for a particular page. How exactly does the MMU know that a virtual to physical map is present in the pagetable or not?

A Level-1 page table entry can be one of four possible types. The 1st type is given below: 
A fault entry that generates an abort exception. This can be either a prefetch or data abort, depending on the type of access. This effectively indicates virtual addresses that are unmapped.
In this case the bit [0] and [1] are set to 0. This is how the MMU identifies that it's a fault entry.
Same is the case with Level-2 page table entry.
Does the Translation Table Base Address (TTBR) register, Level 1 page table and Level 2 page table contain Physical addresses or Virtual addresses?
TTBR: Contain physical address of the pgd base
Level 1 page table (pgd): Physical address pointing to the pte base
Level 2 page table (pte): Physical address pointing to the physical page frame
Since page tables are in kernel space and kernel virtual memory is mapped directly to RAM. Using just an easy macro like __virt_to_phys(), we can get the physical address for the pgd base or pte base or pte entry.


## Kernel Synchronization

* Why do we need synchronization mechanisms in Linux kernel?
* What are the different synchonization mechanisms present in Linux kernel?
* What are the differences between spinlock and mutex?
* What is lockdep?
* Which synchronization mechanism is safe to use in interrupt context and why?
* Explain about the implementation of spinlock in case of ARM architecture.
* Explain about the implementation of mutex in case of ARM architecture.
* Explain about the notifier chains.
* Explain about RCU locks and when are they used?
* Explain about RW spinlocks locks and when are they used?
* Which are the synchronization technoques you use 'between processes', 'between processe and interrupt' and 'between interrupts'; why and how ?
* What are the differences between semaphores and spinlocks?

## Process Management and Process Scheduling
What are the different schedulers class present in the linux kernel?
How to create a new process?
What is the difference between fork( ) and vfork( )?
Which is the first task what is spawned in linux kernel?
What are the processes with PID 0 and PID 1?
PID 0 - idle task
PID 1 - init 
How to extract task_struct of a particular process if the stack pointer is given?
How does scheduler picks particular task?
When does scheduler picks a task?
How is timeout managed?
How does load balancing happens?
Explain about any scheduler class?
Explain about wait queues and how they implemented? Where and how are they used?
What is process kernel stack and process user stack? What is the size of each and how are they allocated?
Why do we need seperate kernel stack for each process?
What all happens during context switch?
What is thread_info? Why is it stored at the end of kernel stack?
What is the use of preempt_count variable?
What is the difference between interruptible and uninterruptible task states?
How processes and threads are created? (from user level till kernel level)
How is virtual run time (vruntime) calculated?

                                      Timers and Time Management
What are jiffies and HZ?
What is the initial value of jiffies when the system has started?
Explain about HR timers and normal timers?
On what hardware timers, does the HR timers are based on?
How to declare that a specific hardware timers is used for kernel periodic timer interrupt used by the scheduler?
How software timers are implemented?

                                       Power Management in Linux
Explain about cpuidle framework.
Explain about cpufreq framework.
Explain about clock framework.
Explain about regulator framework.
Explain about suspened and resume framwork.
Explain about early suspend and late resume.
Explain about wakelocks.

                                          Linux Kernel Modules
How to make a module as loadable module?
How to make a module as in-built module?
Explain about Kconfig build system?
Explain about the init call mechanism.
What is the difference between early init and late init?
Early init:
Early init functions are called when only the boot processor is online.
Run before initializing SMP.
Only for built-in code, not modules.
Late init:
Late init functions are called _after_ all the CPUs are online.

                                         Linux Kernel Debugging

What is Oops and kernel panic?
Does all Oops result in kernel panic?
What are the tools that you have used for debugging the Linux kernel?
What are the log levels in printk?
Can printk's be used in interrupt context?
How to print a stack trace from a particular function?
What's the use of early_printk( )?
Explan about the various gdb commands.

                                                  Miscellaneous

How are the atomic functions implemented in case of ARM architecture?
How is container_of( ) macro implemented? 
Explain about system call flow in case of ARM Linux.
What 's the use of __init and __exit macros?
How to ensure that init function of a partiuclar driver was called before our driver's init function is called (assume that both these drivers are built into the kenrel image)?
What's a segementation fault and what are the scenarios in which segmentation fault is triggered?
If the scenarios which triggers the segmentation fault has occurred, how the kernel identifies it and what are the actions that the kernel takes? 

                                      Process management
1) how to manipulate the current process.
2) what are kernel thread.
3) how threads are implemented in linux kernel.
4) What are different state of a process in lunix.
5) what is difference between process and thread.
6) generally what resources are shared between threads.
7) what is process descriptor
8) what is task_struct.
9) what is therad_info structure for.
10) what was the need of thread_info structure.
11) difference betwen fork() and vfork()
12) what is process context.
13) what is zombie process.
14) how parent less process is handles in linux.
                                              Process Scheduling
1) what is process scheduling
2) what is cooperative multitasking and pre-emptive multitasking.
3) what is yielding.
4) what is limitation of cooperative multitasking.
5) I/O bound versus Processor bound process.
6) what is process priority.
7) What kind of priority is maintained in linux.
8) what is nice value.
9) what is virtual run time.
10) what are the available scheduling classes in linux.
11) which type os scheduling used in linux.
12) how next task is picked for scheduling.
13) what is scheduler entry point in linux.
14) what is waitqueus.
15) How context switching is handled in linux.
16) what is user preemption and kernel preemption
                                                    Syscalls
1) what is syscalls.
2) how system calls are implemented in linux.
3) what happens when process in userspace calls a syscall.
4) what is the need of verifying parameter in definition of syscall.
5) what is system calls context.
6) why it is not recommended to writing new syscall.