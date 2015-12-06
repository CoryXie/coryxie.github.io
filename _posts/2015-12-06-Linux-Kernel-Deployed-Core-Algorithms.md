---
layout: post
title: "Core Algorithms Deployed in Linux Kernel"
description: 
headline: 
modified: 2015-12-05
category: ProgrammingTips
tags: [Programming]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

Algorithms that are the main driver behind a system are easier to find in non-algorithms courses for the same reason theorems with immediate applications are easier to find in applied mathematics rather than pure mathematics courses. It is rare for a practical problem to have the exact structure of the abstract problem in a lecture. The value of these courses is learning that there are intricate ways to exploit the structure of a problem to find efficient solutions. Advanced algorithms is also where one meets simple algorithms whose analysis is non-trivial. This blog post tries to understand some Basic Data Structures and Algorithms in the Linux kernel.

# Single Linked List

The file [`lib/llist.c`](https://github.com/torvalds/linux/blob/master/lib/llist.c) and [`include/linux/llist.h`](https://github.com/torvalds/linux/blob/master/include/linux/llist.h) implements *Lock-less NULL terminated single linked list*.

The basic data structures of a single linked list:

```c

	struct llist_head {
	        struct llist_node *first;
	};
	
	struct llist_node {
	        struct llist_node *next;
	};

	#define LLIST_HEAD_INIT(name)   { NULL }
	#define LLIST_HEAD(name)        struct llist_head name = LLIST_HEAD_INIT(name)

```

The following is the APIs provided by this library.

```c

	#define llist_entry(ptr, type, member)
	#define llist_for_each(pos, node)
	#define llist_for_each_entry(pos, node, member)
	#define llist_for_each_entry_safe(pos, n, node, member)
	
	static inline void init_llist_head(struct llist_head *list);
	static inline bool llist_empty(const struct llist_head *head);
	static inline bool llist_add(struct llist_node *new, struct llist_head *head);
	static inline struct llist_node *llist_del_all(struct llist_head *head);
	struct llist_node *llist_del_first(struct llist_head *head);
	struct llist_node *llist_reverse_order(struct llist_node *head);
	bool llist_add_batch(struct llist_node *new_first, struct llist_node *new_last,
                     struct llist_head *head);

```

Note that the `llist_add()` adds the new node in the first of the node list, so the `llist_del_first()` is actually deleting the newest added node (and returns this deleted node). Also note that the `llist_add()` is in fact calling `return llist_add_batch(new, new, head)` to do the actual work. 

The `llist_add_batch()` is doing lock-less programming with the use of `cmpxchg()`. It sticks (with the `while loop`) to make sure the `head->first` is updated to point to the `new_first`, while at the same time the `new_last->next` points to the original `head->first`.

```c

	bool llist_add_batch(struct llist_node *new_first, struct llist_node *new_last,
	                     struct llist_head *head)
	{
	        struct llist_node *first;
	
	        do {
	                new_last->next = first = ACCESS_ONCE(head->first);
	        } while (cmpxchg(&head->first, first, new_first) != first);
	
	        return !first;
	}

```

The `llist_del_all()` is a simple lock-less programming by using `return xchg(&head->first, NULL)` to replace `head->first` with a `NULL` pointer, and returning the original `head->first` (which is the list of nodes in this single linked list).

The `llist_del_first()` is also doing lock-less programming with `cmpxchg()`. It sticks (with the `while loop`) to make sure the `head->first` is updated to point to the original `head->first->next` (then reuturns the original `head->first` which is the `first` of the node list). However, `llist_del_first()` must be used in some restricted manner. Only one `llist_del_first()` user can be used simultaneously with multiple `llist_add()` users without lock.  Because otherwise `llist_del_first()`, `llist_add()`, `llist_add()` (or `llist_del_all()`, `llist_add()`, `llist_add()`) sequence in another user may change `head->first->next`, but keep `head->first`.  If multiple consumers are needed, we have to use `llist_del_all()` or use lock between consumers.

```c

	struct llist_node *llist_del_first(struct llist_head *head)
	{
	        struct llist_node *entry, *old_entry, *next;
	
	        entry = smp_load_acquire(&head->first);
	        for (;;) {
	                if (entry == NULL)
	                        return NULL;
	                old_entry = entry;
	                next = READ_ONCE(entry->next);
	                entry = cmpxchg(&head->first, old_entry, next);
	                if (entry == old_entry)
	                        break;
	        }
	
	        return entry;
	}

```

Note the list entries can not be traversed safely before being deleted from the list. The list entries deleted via `llist_del_all()` can be traversed with traversing function such as `llist_for_each()` etc. The order of deleted entries is from the newest to the oldest added one. If you want to traverse from the oldest to the newest, you must reverse the order by yourself before traversing with the `llist_reverse_order()` function (reverse the order of a chain of llist entries and return the new first entry).

```c

	struct llist_node *llist_reverse_order(struct llist_node *head)
	{
	        struct llist_node *new_head = NULL;
	
	        while (head) {
	                struct llist_node *tmp = head;
	                head = head->next;
	                tmp->next = new_head;
	                new_head = tmp;
	        }
	
	        return new_head;
	}

```

1) `new_head` is initially set to NULL.

2) While `head` is not NULL, the `while` loop continues.
 
3) The `tmp` pointer points to current `head`, then update `head` to `head->next`;
 
4) The `tmp->next` is updated to point to `new_head`. In the 1st loop `new_head` is NULL, so it effectively makes `tmp->next` to be NULL. Since `new_head` will be updated (in next step) to point to previous (current) `tmp` which is the current `head` in this current loop, this makes the `tmp` to `point back` to the previous `head` (thus the reversing occurs).
 
5) Then `new_head` is updated to point to `tmp`. This moves the `new_head` to next node to be used as the new head. 
 
6) The `while` loop goes back and if `head` (which has been updated to the next node in the original list) is not NULL, it continues.

The following is a complete run of a single linked list with 4 nodes. We can see the key fact is that both `head` and `tmp` move from the first node toward the last node, while `new_head` goes behind the move of `head` and `tmp` for one node. During the time `tmp->next = new_head` makes the link go in reverse order.

<img src="{{ site.baseurl }}/images/2015-12-06-1/llist_reverse_order1.png" alt=llist_reverse_order 1">
<img src="{{ site.baseurl }}/images/2015-12-06-1/llist_reverse_order2.png" alt=llist_reverse_order 2">
<img src="{{ site.baseurl }}/images/2015-12-06-1/llist_reverse_order3.png" alt=llist_reverse_order 3">

The `include/linux/llist.h` has the following file comments.

```c

	/*
	 * Lock-less NULL terminated single linked list
	 *
	 * If there are multiple producers and multiple consumers, llist_add
	 * can be used in producers and llist_del_all can be used in
	 * consumers.  They can work simultaneously without lock.  But
	 * llist_del_first can not be used here.  Because llist_del_first
	 * depends on list->first->next does not changed if list->first is not
	 * changed during its operation, but llist_del_first, llist_add,
	 * llist_add (or llist_del_all, llist_add, llist_add) sequence in
	 * another consumer may violate that.
	 *
	 * If there are multiple producers and one consumer, llist_add can be
	 * used in producers and llist_del_all or llist_del_first can be used
	 * in the consumer.
	 *
	 * This can be summarized as follow:
	 *
	 *           |   add    | del_first |  del_all
	 * add       |    -     |     -     |     -
	 * del_first |          |     L     |     L
	 * del_all   |          |           |     -
	 *
	 * Where "-" stands for no lock is needed, while "L" stands for lock
	 * is needed.
	 *
	 * The list entries deleted via llist_del_all can be traversed with
	 * traversing function such as llist_for_each etc.  But the list
	 * entries can not be traversed safely before deleted from the list.
	 * The order of deleted entries is from the newest to the oldest added
	 * one.  If you want to traverse from the oldest to the newest, you
	 * must reverse the order by yourself before traversing.
	 *
	 * The basic atomic operation of this list is cmpxchg on long.  On
	 * architectures that don't have NMI-safe cmpxchg implementation, the
	 * list can NOT be used in NMI handlers.  So code that uses the list in
	 * an NMI handler should depend on CONFIG_ARCH_HAVE_NMI_SAFE_CMPXCHG.
	 */

```

The `kernel/smp.c` has an exmaple usage of this API.

```c
	
	/**
	 * flush_smp_call_function_queue - Flush pending smp-call-function callbacks
	 *
	 * @warn_cpu_offline: If set to 'true', warn if callbacks were queued on an
	 *                    offline CPU. Skip this check if set to 'false'.
	 *
	 * Flush any pending smp-call-function callbacks queued on this CPU. This is
	 * invoked by the generic IPI handler, as well as by a CPU about to go offline,
	 * to ensure that all pending IPI callbacks are run before it goes completely
	 * offline.
	 *
	 * Loop through the call_single_queue and run all the queued callbacks.
	 * Must be called with interrupts disabled.
	 */
	static void flush_smp_call_function_queue(bool warn_cpu_offline)
	{
	        struct llist_head *head;
	        struct llist_node *entry;
	        struct call_single_data *csd, *csd_next;
	        static bool warned;
	
	        WARN_ON(!irqs_disabled());
	
	        head = this_cpu_ptr(&call_single_queue);
	        entry = llist_del_all(head);
	        entry = llist_reverse_order(entry);
	
	        /* There shouldn't be any pending callbacks on an offline CPU. */
	        if (unlikely(warn_cpu_offline && !cpu_online(smp_processor_id()) &&
	                     !warned && !llist_empty(head))) {
	                warned = true;
	                WARN(1, "IPI on offline CPU %d\n", smp_processor_id());
	
	                /*
	                 * We don't have to use the _safe() variant here
	                 * because we are not invoking the IPI handlers yet.
	                 */
	                llist_for_each_entry(csd, entry, llist)
	                        pr_warn("IPI callback %pS sent to offline CPU\n",
	                                csd->func);
	        }
	
	        llist_for_each_entry_safe(csd, csd_next, entry, llist) {
	                smp_call_func_t func = csd->func;
	                void *info = csd->info;
	
	                /* Do we wait until *after* callback? */
	                if (csd->flags & CSD_FLAG_SYNCHRONOUS) {
	                        func(info);
	                        csd_unlock(csd);
	                } else {
	                        csd_unlock(csd);
	                        func(info);
	                }
	        }
	
	        /*
	         * Handle irq works queued remotely by irq_work_queue_on().
	         * Smp functions above are typically synchronous so they
	         * better run first since some other CPUs may be busy waiting
	         * for them.
	         */
	        irq_work_run();
	}

```

# Doubly Linked List

TODO

# B+ Trees

B+ Trees with comments telling you what you can't find in the textbooks.

A relatively simple B+Tree implementation. I have written it as a learning exercise to understand how B+Trees work. Turned out to be useful as well.

...

A tricks was used that is not commonly found in textbooks. The lowest values are to the right, not to the left. All used slots within a node are on the left, all unused slots contain NUL values. Most operations simply loop once over all slots and terminate on the first NUL.

# Priority sorted lists

Priority sorted lists used for mutexes, drivers, etc.

# Red-Black trees

Red-Black trees are used for scheduling, virtual memory management, to track file descriptors and directory entries,etc.

# Interval trees

TODO

# Radix trees

Radix trees, are used for memory management, NFS related lookups and networking related functionality.

A common use of the radix tree is to store pointers to struct pages;

# Priority heap

Priority heap, which is literally, a textbook implementation, used in the control group system.

Simple insertion-only static-sized priority heap containing pointers, based on CLR, chapter 7

# Hash functions

Hash functions, with a reference to Knuth and to a paper.

Knuth recommends primes in approximately golden ratio to the maximum integer representable by a machine word for multiplicative hashing. Chuck Lever verified the effectiveness of this technique:

http://www.citi.umich.edu/techreports/reports/citi-tr-00-1.pdf

These primes are chosen to be bit-sparse, that is operations on them can use shifts and additions instead of multiplications for machines where multiplications are slow.


# Hash function using a Rotating Hash algorithm

Some parts of the code, such as this driver, implement their own hash function.

Knuth, D. The Art of Computer Programming, Volume 3: Sorting and Searching, Chapter 6.4. Addison Wesley, 1973
Hash tables used to implement inodes, file system integrity checks etc.
Bit arrays, which are used for dealing with flags, interrupts, etc. and are featured in Knuth Vol. 4.

# Binary search

Binary search is used for interrupt handling, register cache lookup, etc.

# Binary search with B-trees

TODO

# Depth first search and variant used in directory configuration.

Performs a modified depth-first walk of the namespace tree, starting (and ending) at the node specified by start_handle. The callback function is called whenever a node that matches the type parameter is found. If the callback function returns a non-zero value, the search is terminated immediately and this value is returned to the caller.

# Breadth first search

Breadth first search is used to check correctness of locking at runtime.

# Merge sort

Merge sort on linked lists is used for garbage collection, file system management, etc.

# Bubble sort

Bubble sort is amazingly implemented too, in a driver library.

# Knuth-Morris-Pratt string matching,

Implements a linear-time string-matching algorithm due to Knuth, Morris, and Pratt [1]. Their algorithm avoids the explicit computation of the transition function DELTA altogether. Its matching time is O(n), for n being length(text), using just an auxiliary function PI[1..m], for m being length(pattern), precomputed from the pattern in time O(m). The array PI allows the transition function DELTA to be computed efficiently "on the fly" as needed. Roughly speaking, for any state "q" = 0,1,...,m and any character "a" in SIGMA, the value PI["q"] contains the information that is independent of "a" and is needed to compute DELTA("q", "a") 2. Since the array PI has only m entries, whereas DELTA has O(m|SIGMA|) entries, we save a factor of |SIGMA| in the preprocessing time by computing PI rather than DELTA.

# Boyer-Moore pattern matching

Boyer-Moore pattern matching with references and recommendations for when to prefer the alternative.

Implements Boyer-Moore string matching algorithm:

Note: Since Boyer-Moore (BM) performs searches for matchings from right to left, it's still possible that a matching could be spread over multiple blocks, in that case this algorithm won't find any coincidence.

If you're willing to ensure that such thing won't ever happen, use the Knuth-Pratt-Morris (KMP) implementation instead. In conclusion, choose the proper string search algorithm depending on your setting.

Say you're using the textsearch infrastructure for filtering, NIDS or
any similar security focused purpose, then go KMP. Otherwise, if you really care about performance, say you're classifying packets to apply Quality of Service (QoS) policies, and you don't mind about possible matchings spread over multiple fragments, then go BM.

# References

* http://cstheory.stackexchange.com/questions/19759/core-algorithms-deployed