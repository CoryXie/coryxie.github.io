---
layout: post
title: "ZFS-FUSE ARC Cache Theory"
description: 
headline: 
modified: 2015-12-13
category: FileSysDev
tags: [Filesystem]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---


The `Adjustable Replacement Cache (ARC)` of Solaris ZFS is really an interesting piece of software which is based on the `Adaptive Replacement Cache` as described by IBM researchers *Megiddo* and *Modha* in their papers (`One Up on LRU` and `ARC: A Self-Tuning, Low Overhead Replacement Cache`). `Adaptive Replacement Cache (ARC)` is a page replacement algorithm with better performance than `LRU (least recently used)`. This is accomplished by keeping track of both `frequently used` and `recently used` pages plus a `recent eviction history` for both. In 2006, IBM was granted a patent for the adaptive replacement cache policy (`US6996676B2 - System and method for implementing an adaptive replacement cache policy`). ZFS extended it for its own usage with new features, but the basic theory still holds. This blog post tries to understand the theory of the read cache of ZFS, concentrating on how data gets into the cache and how the cache adjust itselfs to the load and thus earns its name as `Adjustable Replacement Cache`.

## Caching Basics and Weakness of Traditional Caches

Computer memory systems generally comprise two memory levels: main (or cache) and auxiliary. Cache memory is faster than auxiliary memory, but is also significantly more expensive. Consequently, the size of the cache memory is usually only a fraction of the size of the auxiliary memory.

Caching is one of the most fundamental metaphors in modern computing. It is widely used in storage systems, databases, web servers, middleware, processors, file systems, disk drives, and operating systems. Memory caching is also used in varied and numerous other applications such as data compression and list updating. As a result a substantial progress in caching algorithms could affect a significant portion of the modern computation stack.

Both cache and auxiliary memories are managed in units of uniformly sized items known as pages. Requests for pages are first directed to the cache. A request for a page is directed to the auxiliary memory only if the page is not found in the cache. In this case, a copy is “paged in” to the cache from the auxiliary memory. This is called “demand paging” and it precludes “pre-fetching” pages from the auxiliary memory to the cache. If the cache is full, one of the existing pages must be paged out before a new page can be brought in.

A replacement policy determines which page is “paged out”. A commonly used criterion for evaluating a replacement policy is its hit ratio, the frequency at which a page is found in the cache as opposed to finding the page in the auxiliary memory. The miss rate is the fraction of pages paged into the cache from the auxiliary memory. The replacement policy goal is to maximize the hit ratio measured over a very long trace while minimizing the memory overhead involved in implementing the policy.

Most current replacement policies remove pages from the cache based on “recency” that is removing pages that have least recently been requested, “frequency” that is removing pages that are not often requested, or a combination of recency and frequency. Certain replacement policies also have parameters that must be carefully chosen or “tuned” to achieve optimum performance.

The replacement policy that provides an upper bound on the achievable hit ratio by any online policy is `Belady's MIN or OPT (MIN)`. However, this approach uses a prior knowledge of the entire page reference stream and is not realizable in practice when the page reference stream is not known ahead of time. `MIN` replaces the page that has the `greatest forward distance`. Given `MIN` as a reference, a replacement policy that automatically adjusts to an observed workload is much preferable.

The most commonly used replacement policy is based on the concept of replace the `least recently used (LRU)` page. The LRU policy focuses solely on recency, always replacing the least recently used page. As one of the original replacement policies, approximations and improvements to LRU abound. If the workload or the request stream is drawn from a LRU `Stack Depth Distribution (SDD)`, then LRU is the optimal policy.

LRU has several advantages: it is relatively simple to implement and responds well to changes in the underlying `Stack Depth Distribution (SDD)` model. However, while the SDD model captures recency, it does not capture frequency. Each page is equally likely to be referenced and stored in cache. Consequently, the LRU model is useful for treating the `clustering effect of locality` but not for treating `non-uniform page referencing`. In addition, the LRU model is vulnerable to `one-time-only sequential read` requests, or scans, that replace higher-frequency pages with pages that would not be requested again, reducing the hit ratio. In other terms, the LRU model is not “`scan resistant`”.

The `Independent Reference Model (IRM)` provides a `workload characterization` that captures the notion of frequency. Specifically, `IRM` assumes that each page reference is drawn in an independent fashion from a fixed distribution over the set of all pages in the auxiliary memory. Under the IRM model, the `least frequently used (LFU)` policy that replaces the least frequently used page is optimal.

While the LFU policy is `scan-resistant`, it presents several drawbacks. The LFU policy requires logarithmic implementation complexity in cache size and pays almost no attention to recent history. In addition, the LFU policy does not adapt well to changing access patterns since it accumulates state pages with high frequency counts that may no longer be useful.

A relatively recent algorithm, `LRU-2`, approximates the LFU policy while eliminating its lack of adaptivity to the evolving distribution of page reference frequencies. The `LRU-2` algorithm remembers, for each page, the last two times that page was requested and discards the page with the least recent penultimate reference. Under the `Independent Reference Model (IRM)` assumption, the `LRU-2` algorithm has the largest expected hit ratio of any online algorithm that knows the two most recent references to each page.

The `LRU-2` algorithm works well on several traces. Nonetheless, `LRU-2` still has two practical limitations:

1. The `LRU-2` algorithm maintains a priority queue, requiring logarithmic implementation complexity.
2. The `LRU-2` algorithm contains one crucial tunable parameter, namely, `Correlated Information Period (CIP)`. CIP roughly captures the amount of time a page seen only once recently should be kept in the cache.

In practice, logarithmic implementation complexity engenders a severe memory overhead. Another algorithm, `2Q`, reduces the implementation complexity to constant per request rather than logarithmic by using a simple LRU list instead of the priority queue used in `LRU-2` algorithm. Otherwise, the 2Q algorithm is similar to the `LRU-2` algorithm.

The choice of the parameter `Correlated Information Period (CIP)` crucially affects performance of the `LRU-2` algorithm. No single fixed a priori choice works uniformly well across various cache sizes. Consequently, a judicious selection of this parameter is crucial to achieving good performance.

Furthermore, no single a priori choice works uniformly well across various workloads and cache sizes. For example, a very small value for the `CIP` parameter works well for stable workloads drawn according to the `Independent Reference Model (IRM)`, while a larger value works well for workloads drawn according to the `Stack Depth Distribution (SDD)`, but no value works well for both. This underscores the need for online, on-the-fly adaptation.

However, the second limitation of the `LRU-2` algorithm persists even in the `2Q` algorithm. The algorithm `2Q` introduces two parameters, `Kin` and `Kout`. The parameter `Kin` is essentially the same as the parameter `CIP` in the `LRU-2` algorithm. Both `Kin` and `Kout` are parameters that need to be carefully tuned and both are sensitive to workload conditions and types.

Another recent algorithm similar to the `2Q` algorithm is `Low Inter-reference Recency Set (LIRS)`. The `LIRS` algorithm maintains a variable size `LRU stack` whose LRU page is the `Llirs`-th page seen at least twice recently, where `Llirs` is a parameter. From all the pages in the stack, the `LIRS` algorithm keeps in the cache all the `Llirs` pages seen at least twice recently as well as the `Llirs` pages seen only once recently.

The parameter `Llirs` is similar to the `CIP` of the `LRU-2` algorithm or `Kin` of `2Q`. Just as the `CIP` affects the `LRU-2` algorithm and `Kin` affects the `2Q` algorithm, the parameter `Llirs` crucially affects the `LIRS` algorithm. A further limitation of `LIRS` is that it requires a certain “`stack pruning`” operation that, in the worst case, may have to touch a very large number of pages in the cache. In addition, the `LIRS` algorithm stack may grow arbitrarily large, requiring a priori limitation. However, with a stack size of twice the cache size, LIRS becomes virtually identical to `2Q` with `Kin=1%` and `Kout=99%`.

Over the past few years, interest has focused on `combining recency and frequency` in various ways, attempting to bridge the gap between LRU and LFU. Two replacement policy algorithms exemplary of this approach are `frequency-based replacement, FBR`, and `least recently/frequently used, LRFU`.

The `frequency-based replacement algorithm, FBR`, maintains a `least recently used (LRU)` list, but divides it into three sections: `new`, `middle`, and `old`. For every page in cache, the FBR algorithm also maintains a counter. On a cache hit, the FBR algorithm moves the hit page to the `most recently used (MRU)` position in the new section. If the hit page was in the middle or the old section, then its `reference count` is incremented. If the hit page was in the new section then the reference count is not incremented; this key concept is “`factoring out locality`”. On a cache miss, the FBR algorithm replaces the page in the old section with the smallest reference count.

One limitation of the FBR algorithm is that the algorithm must `periodically resize (re-scale) all the reference counts` to prevent cache pollution due to stale pages with high reference count but no recent usage. The FBR algorithm also has several tunable parameters: the size of all three sections, and two other parameters `Cmax` and `Amax` that control periodic resizing. Much like the `LRU-2` and `2Q` algorithms, different values of these tunable parameters may be suitable for different workloads or for different cache sizes. The performance of the FBR algorithm is similar to that of the LRU-2 and 2Q algorithms.

Another replacement policy that combines the concepts of recency, LRU, and frequency, LFU, is the `Least Recently/Frequently Used (LRFU)` algorithm. The LRFU algorithm initially assigns a value `C(x)=0` to every page `x`, and, at every time `t`, updates as: `C(x)=1+2−λC(x)` if `x` is referenced at time `t`;  `C(x)=2−λC(x)` otherwise, where `λ` is a tunable parameter.

This update rule is a form of exponential smoothing that is widely used in statistics. The `LRFU` policy is to replace the page with the smallest `C(x)` value. Intuitively, as `λ` approaches 0, the `C` value is simply the number of occurrences of page `x` and LRFU collapses to LFU. As `λ` approaches 1, the `C` value emphasizes recency and the LRFU algorithm collapses to LRU. The performance of the algorithm depends crucially on the choice of λ.

A later adaptive version, the `Adaptive LRFU (ALRFU)` algorithm, dynamically adjusts the parameter `λ`. Still, the LRFU the LRFU algorithm has two fundamental limitations that hinder its use in practice:

1. `LRFU` and `ALRFU` both require an additional tunable parameter for controlling correlated references. The choice of this parameter affects performance of the replacement policy.
2. The implementation complexity of `LRFU` fluctuates between constant and logarithmic in cache size per request.

However, the practical complexity of the `LRFU` algorithm is significantly higher than that of even the `LRU-2` algorithm. For small values of `λ`, the `LRFU` algorithm can be as much as 50 times slower than LRU. Such overhead can potentially wipe out the entire benefit of a higher hit ratio.

Another replacement policy behaves as an `expert master policy` that simulates a number of caching policies. At any given time, the master policy adaptively and dynamically chooses one of the competing policies as the “winner” and switches to the winner. Rather than develop a new caching policy, the master policy selects the best policy amongst various competing policies. From a practical standpoint, a limitation of the master policy is that it must simulate all competing policies, consequently requiring high space and time overhead.

What is therefore needed is a replacement policy with a high hit ratio and low implementation complexity. Real-life workloads possess a great deal of richness and variation and do not admit a one-size-fits-all characterization. They may contain long sequential I/Os or moving hot spots. The frequency and scale of temporal locality may also change with time. They may fluctuate between stable repeating access patterns and access patterns with transient clustered references. No static, a priori fixed replacement policy will work well over such access patterns. Thus, the need for a cache replacement policy that adapts in an online, on-the-fly fashion to such dynamically evolving workloads while performing with a high hit ratio and low overhead has heretofore remained unsatisfied.

In summary, the standard LRU like mechanisms of other file system caches have some shortfalls. For example, they are not `scan-resistant`. When you read a large amount of blocks sequential, they tend to fill up the cache, even when they are read just once. When the cache is full and you want to place new data in the cache, the least recently used page is evicted from the cache (thrown out of it). In the case of such large sequential reads, the cache would contain only those reads and not really frequently used data. When the large sequential reads are just used once, the cache is filled with worthless data from the perspective of a cache.

There is another challenge: A cache can optimize for recency (bycaching the most recently used pages) or for frequency (by caching the most frequently used pages). Neither way is optimal for every workload. Thus a good cache design is able to optimize itself. 

## Theory for Implementing an Adaptive Replacement Cache Policy

FIG. 1 illustrates an exemplary high-level architecture of a computer memory system 100 comprising an adaptive replacement cache policy system 10 that utilizes, a cache 15 and an auxiliary memory 20. 

<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00001.png" alt="ZFS-FUSE ARC US06996676 D00001">

The design of system 10 presents a new replacement policy. This replacement policy manages twice the number of pages present in cache 15 (also referred to herein as DBL(2c)). System 10 is derived from a fixed replacement policy that has a tunable parameter. The extrapolation to system 10 transforms the tunable parameter to one that is automatically adjusted by system 10.

The present cache replacement policy DBL(2c) manages and remembers twice the number of pages present in the cache 15, where `c` is the number of pages in a typical cache 15. As seen in FIG. 2, the cache replacement policy DBL(2c) maintains two variable-sized lists L1 205 and L2 210.

List L1 205 contains pages requested only once recently, and establishes the recency aspect of page requests. List L2 210 contains pages requested at least twice recently, and establishes the frequency aspect of page requests. The pages are sorted in each list from most recently used, MRU, to least recently used, LRU, as shown by the arrows in list L1 205 and list L2 210.

The present cache replacement policy DBL(2c) replaces the LRU page in list L1 205 if list L1 205 contains exactly c pages; otherwise, it replaces the LRU page in list L2 210. The method of operation 300 for the cache replacement policy DBL(2c) is further shown FIG. 3. The policy attempts to keep both lists L1 and L2 to contain roughly c pages.

<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00002.png" alt="ZFS-FUSE ARC US06996676 D00002">

Given that a page X is requested at block 302, the cache replacement policy DBL(2c) first determines at decision block 305 whether input page X exists in list L1 205. If so, then page X has recently been seen once, and is moved from the recency list, L1 205, to the frequency list, L2 210. The cache replacement policy DBL(2c) deletes page X from list L1 205 at block 310 and moves page X to the top of list L2 210 at block 315.

Page X is now the most recently requested page in list L2 210, so it is moved to the top of this list. At block 320, the cache replacement policy DBL(2c) of system 10 updates the number of pages in each list as shown, where l1 is the number of pages in list L1 205, and l2 is the number of pages in list L2 210. The total number of pages in the cache replacement policy DBL(2c) is still at most 2c, since a page was simply moved from list L1 205 to list L2 210.

If at decision block 305 page X was not found in list L1 205, the cache replacement policy DBL(2c) determines, at decision block 325, if page X is in list L2 210. If so, page X is now the most recently requested page in list L2 210 and the cache replacement policy DBL(2c) moves it to the top of the list at block 330. If page X is in neither list L1 205 nor list L2 210, it is a miss and the cache replacement policy DBL(2c) must decide where to place page X.

The sizes of the two lists can fluctuate, but the cache replacement policy DBL(2c) wishes to maintain, as closely as possible, the same number of pages in list L1 205 and list L2 210, maintaining the balance between recency and frequency. If there are exactly c pages in list L1 205 at decision block 335, the cache replacement policy DBL(2c) deletes the least recently used (LRU) page in list L1 205 at block 340, and makes page X the most recently used (MRU) page in list L1 205 at block 345.

If the number of pages l1 in list L1 205 is determined at decision block 335 to be less than c, the cache replacement policy DBL(2c) determines at decision block 350 if the cache 15 is full, i.e., whether l1+l2=2c. If not, the cache replacement policy DBL(2c) inserts page X as the MRU page in list L1 205 at block 355, and adds one to l1, the number of pages in L1 205, at block 360. If the cache 15 is determined to be full at decision block 350, the cache replacement policy DBL(2c) deletes the LRU page in list L2 210 at block 365 and subtracts one from l2, the number of pages in list L2 210.

Having made room for a new page, the cache replacement policy DBL(2c) then proceeds to blocks 355 and 360, inserting X as the MRU page in L1 205 and adding one to l1, the number of pages in list L1 205. Pages can only be placed in list L2 210, the frequency list, by moving them from list L1 205, the recency list. New pages are always added to list L1 205.

The method 300 of system 10 is based on the following code outline:

```c

        if (L1->hit(page)) {	
           L1->delete(page);
           L2->insert_mru(page);
         }
        else if (L2->hit(page)) {
           L2->delete(page);
           L2->insert_mru(page);
         }
        else if (L1->length() == c) {
           L1->delete_lru();
           L1->insert_mru(page);
         }
        else {
           if (L1->length() + L2->length() == 2*c) {
              L2->delete_lru();
           }
           L1->insert_mru(page);
         }

```

Based on the performance of the cache replacement policy DBL(2c) in method 300 of FIG. 3, it can be seen that even though the sizes of the two lists L1 205 and L2 210 fluctuate, the following is always true: 


>- 0 < l1 + l2 <= 2c; 
>- 0 <= l1 <= c; and 
>- 0 <= l2 <= 2c.

In addition, the replacement decisions of the cache replacement policy DBL(2c) at blocks 335 and 350 equalize the sizes of two lists. System 10 is based on method 300 shown of FIG. 3. System 10 contains demand paging policies that track all 2c items that would have been in a cache 15 of size 2c managed by the cache replacement policy DBL(2c), but physically keeps only (at most) c of those pages in the cache 15 at any given time.

<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00003.png" alt="ZFS-FUSE ARC US06996676 D00003">

With further reference to FIG. 4, system 10 introduces the concept of a “dynamic” or “sliding” window 425. To this end, the window 425 has a capacity c, and divides the list L1 into two dynamic portions B1 410 and T1 405, and further divides the list L2 into two dynamic portions B2 420 and T2 415. These dynamic list portions meet the following conditions:

1) List portions T1 405 and B1 410 are disjoint, as are list portions T2 415 and B2 420.

2) List L1 205 is comprised of list portions B1 410 and T1 405, as follows: 

>- L1 205 = [T1 405 U B1 410]

3) List L2 210 is comprised of list portions B2 420 and T2 415, as follows: 

>- L2 210 = [T2 415 U B2 420].

4) If the number of pages l1+l2 in lists L1 and L2 is less than c, then the list portions B1 410 and B2 420 are empty, as expressed by the following expression:

>- If |L1 205|U|L2 210|<c, then both B1 410 and B2 420 are empty.

5) If the number of pages l1+l2 in lists L1 and L2 is greater than, or equal to c, then the list portions T1 405 and T2 415 together contain exactly c pages, as expressed by the following expression:

>- If |L1 205|U|L2 210|≧c, then T1 405 and T2 415 contain exactly c pages.

6) Either list portion T1 405 is empty or list portion B1 410 is empty or the LRU page in list portion T1 405 is more recent than the MRU page in list portion B1 410. Similarly, either list portion T2 415 is empty or list portion B2 420 is empty or the LRU page in list portion T2 415 is more recent than the MRU page in list portion B2 420. In plain words, every page in T1 is more recent than any page in B1 and every page in T2 is more recent than any page in B2.

7) For all traces and at each time, pages in both list portions T1 405 and T2 415 are exactly the same pages that are maintained in cache 15.

The foregoing conditions imply that if a page in list portion L1 205 is kept, then all pages in list portion L1 205 that are more recent than this page must also be kept in the cache 15. Similarly, if a page in list portion L2 210 is kept, then all pages in list portion L2 210 that are more recent than this page must also be kept in the cache 15. Consequently, the cache replacement policy that satisfies the above seven conditions “skims the top (or most recent) few pages” in list portion L1 205 and list portion L2 210.

If a cache 15 managed by the cache replacement policy of system 10 is full, that is if: `|T1|U|T2|=c`, then it follows from the foregoing conditions that, for any trace, on a cache 15 miss only two actions are available to the cache replacement policy:

1) either replace the LRU page in list portion T1 405, or
2) replace the LRU page in list portion T2 415. 

The pages in `|T1|U|T2|` are maintained in the cache 15 and a directory; and are represented by the window 425. The pages in list portions B1 410 and B2 420 are maintained in the directory only and not in the cache.

With reference to FIG. 3, the “c” most recent pages will always be contained in the cache replacement policy DBL(2c), which, in turn, deletes either the LRU item in list L1 205 (block 340) or the LRU item in list L2 210 (block 365). In the first case, list L1 205 must contain exactly c items (block 335), while in the latter case, list L2 210 must contain at least c items (bock 350). Hence, the cache replacement policy DBL(2c) does not delete any of the most recently seen c pages, and always contains all pages contained in a LRU cache 15 with c items. Consequently, there exists a dynamic partition of lists L1 205 and L2 210 into list portions T1 405, B1 410, T2 415, and B2 420, such that the foregoing conditions are met.

The choice of 2c as the size of the cache 15 directory for the cache replacement policy DBL(2c) will now be explained. If the cache replacement policy DBL(2c′) is considered for some positive integer c′<c, then the most recent c pages need not always be in the cache replacement policy DBL(2c′). For example, consider the trace 
`1,2, . . . ,c,1,2, . . . c, . . . ,1,2, . . . ,c . . . .` . For this trace, the hit ratio of LRU(c) approaches 1 as the size of the trace increases, but the hit ratio of the cache replacement policy DBL(2c′), for any c′<c, is zero.

The design of the cache replacement policy DBL(2c) can be expanded to a replacement policy FRCp(c) for fixed replacement cache. This policy FRCp(c) has a tunable or self-adjusting parameter p, where 0<p≦c, and satisfies the foregoing seven conditions. In addition, the policy FRCp(c) satisfies a crucial new condition, namely to keep exactly p pages in the list portion T1 405 and exactly (c−p) pages in the list portion T2 415. In other terms, the policy FRCp(c) attempts to keep exactly the MRU p top pages from the list portion L1 205 and the MRU (c−p) top pages from the list portion L2 210 in the cache 15, wherein p is the target size for the list.

The replacement policy FRCp(c) is expressed as follows:

1) If |T1 405|>p, replace the LRU page in list portion T1 405.

2) If |T1 405|<p, replace the LRU page in list portion T2 415.

3) If |T1 405|=p and the missed page is in list portion B1 410, replace the LRU page in list portion T2 415. Similarly, if list portion |T2 405|=p and the missed page is in list portion B2 420, replace the LRU page in list portion T1 405. Replacement decision 3 above can be optional or it can be varied if desired.

System 10 is an adaptive replacement policy based on the design of the replacement policy FRCp(c). At any time, the behavior of system 10 is described once a certain adaptation parameter pε[0, c] is known. For a given value of the parameter p, system 10 behaves exactly as the replacement policy FRCp(c). However, unlike the replacement policy FRCp(c), system 10 does not use a single fixed value for the parameter p over the entire workload. System 10 continuously adapts and tunes p in response to the observed workload.

System 10 dynamically detects, in response to an observed workload, which item to replace at any given time. Specifically, on a cache miss, system 10 adaptively decides whether to replace the LRU page in list portion T1 405 or to replace the LRU page in list portion T2 415, depending on the value of the adaptation parameter p at that time. The adaptation parameter p is the target size for the list portion T1 405. A preferred embodiment for dynamically tuning the parameter p is now described.

Method 500 of system 10 is described by the logic flowchart of FIG. 5 (FIGS. 5A, 5B, 5C, 5D). At block 502, a page X is requested from cache 15. System 10 determines at decision block 504 if page X is in (T1 405 U T2 415). If so, then page X is already in cache 15, a hit has occurred, and at block 506 system 10 moves page X to the top of list portion T2 415, the MRU position in the frequency list.

<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00004.png" alt="ZFS-FUSE ARC US06996676 D00004">

If however, the result at block 504 is false, system 10 ascertains whether page X is in list portion B1 410 at block 508. If so, a miss has occurred in cache 15 and a hit has occurred in the recency directory of system 10. In response, system 10 updates the value of the adaptation parameter, p, at block 510, as follows: 

>- p=min{c,p+max{|B2|/|B1|,1}}, 

where |B2| is the number of pages in the list portion B2 420 directory and |B1| is the number of pages in the list portion B1 410 directory.

System 10 then proceeds to block 512 and moves page X to the top of list portion T2 415 and places it in cache 15. Page X is now at the MRU position in list portion T2 415, the list that maintains pages based on frequency. At decision block 514, system 10 evaluates |T1 405|>p. If the evaluation is true, system 10 moves the LRU page of list portion T1 405 to the top of list portion B1 410 and removes that LRU page from cache 15 at block 516. The LRU page in the recency portion of cache 15 has moved to the MRU position in the recency directory.

Otherwise, if the evaluation at step 514 is false, system 10 moves the LRU page of list portion T2 415 to the top of list portion B2 420 and removes that LRU page from cache 15 at block 518. In this case, the LRU page of the frequency portion of cache 15 has moved to the MRU position in the frequency directory. System 10 makes these choices to balance the sizes of list portion L1 205 and list portion L2 210 while adapting to meet workload conditions.

Returning to decision block 508, if page X is not in B1, system 10 continues to decision block 520 (shown in FIG. 5B) to evaluate if page X is in B2. If this evaluation is true, a hit has occurred in the frequency directory of system 10. System 10 proceeds to block 522 and updates the value of the adaptation parameter, p, as follows: 

>- p=max{0,p−max{|B1|/|B2|,1}} 

where |B2| is the number of pages in the list portion B2 420 directory and |B1| is the number of pages in the list portion B1 410 directory. System 10 then, at block 524, moves page X to the top of list portion T2 415 and places it in cache 15. Page X is now at the MRU position in list portion T2 415, the list that maintains pages based on frequency.

<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00005.png" alt="ZFS-FUSE ARC US06996676 D00005">

System 10 must now decide which page to remove from cache 15. At decision block 526, system 10 evaluates |T1 405|≧max {p,1}. If the result is true, system 10 moves the LRU page of list portion T1 405 to the top of list portion B1 410 and removes that LRU page from cache 15 at block 528. Otherwise, system 10 moves the LRU page of list portion T2 415 to the top of list portion B2 420 and removes that LRU page from cache 15 at block 530.

If at decision block 520 X is not in B2 420, the requested page is not in cache 15 or the directory. More specifically, the requested page is a system miss. System 10 then must determine which page to remove from cache 15 to make room for the requested page. Proceeding to FIG. 5C, system 10 evaluates at decision block 532 |L1|=c. If the result is true, system 10 then evaluates at decision block 534 |T1|<c.


<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00006.png" alt="ZFS-FUSE ARC US06996676 D00006">

If the result of the evaluation at block 534 is false, then system 10 deletes the LRU page of list portion T1 405 and removes it from cache 15, block 536. System 10 then puts the requested page X at the top of list portion T1 405 and places it in cache 15 at block 538.

Returning to decision block 534, if the result is true, system 10 proceeds to block 540 and deletes the LRU page of list portion B1 410. At decision block 542, system 10 evaluates |T1|≧max {p,1}. If the result is false, system 10 moves the LRU page of list portion T2 415 to the top of list portion B2 420 and removes that LRU page from cache 15 at block 544. System 10 then puts the requested page X at the top of list portion T1 405 and places it in cache 15 at block 538.

If the result at decision block 542 is true, system 10 moves the LRU page of list portion T1 405 to the top of list portion B1 410 and removes that LRU page from cache 15 at block 546. System 10 then puts the requested page X at the top of list portion T1 405 and places it in cache 15 at block 538.

Returning now to decision block 532, if the result is false, system 10 proceeds to decision block 548 and evaluates the following condition: 

>- |L1 205|<c and |L1 205|+|L2 210|≧c. 

If the result is false, system 10 puts the requested page X at the top of list portion T1 405 and places it in cache 15 at block 538. If, however, the result is true, system 10 proceeds to decision block 550 (FIG. 5D) and evaluates |L1|+|L2|=2c. If the result is true, system 10 deletes the LRU page of list portion B2 420 at block 552. After this the system proceeds to decision block 556.

<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00007.png" alt="ZFS-FUSE ARC US06996676 D00007">

If the result at decision block 550 is false, system 10 evaluates |T1|≧max {p,1} at decision block 556. If the result is true, system 10 moves the LRU page of list portion T1 405 to the top of list portion B1 410, and removes that LRU page from cache 15 at block 558. System 10 then places the requested page X at the top of list portion T1 405 and places it in cache 15 at block 554. If the result at decision block 556 is false, system 10 moves the LRU page in list portion T2 415 to the top of list portion B2 420 and removes that LRU page from cache 15 at block 560. System 10 then places the requested page X at the top of list portion T1 405 and places it in cache 15 at block 554.

System 10 continually revises the parameter p in response to a page request miss or in response to the location of a hit for page x within list portion T1 405, list portion T2 415, list portion B2 410, or list portion B2 420. The response of system 10 to a hit in list portion B2 410 is to increase the size of T1 405. Similarly, if there is a hit in list portion B2 420, then system 10 increases the size of list portion T2 415. Consequently, for a hit on list portion B1 410 system 10 increases p, the target size of list portion T1 405; a hit on list portion B2 420 decreases p. When system 10 increases p, the size of list portion T1 405, the size of list portion T2 415 (c−p) implicitly decreases.

The precise magnitude of the revision in p is important. The precise magnitude of revision depends upon the sizes of the list portions B1 410 and B2 420. On a hit in list portion B1 410, system 10 increments p by:

>- max{|B2|/|B1|,1}

subject to the cap of c, where |B2| is the number of pages in the list portion B2 420 directory and |B1| is the number of pages in the list portion B1 410 directory; the minimum revision is by 1 unit. Similarly, on a hit in list portion B2 420, system 10 decrements p by:

>- min{|B1|/|B2|,1}

subject to the floor of zero, where |B2| is the number of pages in the list portion B2 420 directory and |B1| is the number of pages in the list portion B1 410 directory; the minimum revision is by 1 unit.

If there is a hit in list portion B1 410, and list portion B1 410 is very large compared to list portion B2 420, then system 10 increases p very little. However, if list portion B, 410 is small compared to list portion B2 420, then system 10 increases p by the ratio |B2|/|B1|. Similarly, if there is a hit in list portion B2 420, and list portion B2 420 is very large compared to list portion B1 410, then system 10 increases p very little. However, if list portion B2 420 is small compared to list portion B1 410, then system 10 increases p by the ratio |B1|/|B2|. In effect, system 10 invests cache 15 resources in the list portion that is receiving the most hits.

Turning now to FIG. 6, the compound effect of a number of such small increments and decrements to p, induces a “random walk” on the parameter p. In effect, the window 425 slides up and down as the sizes of list portions T1 405 and T2 415 change in response to the workload. The window 425 is the number of pages in actual cache 15 memory.

<img src="{{ site.baseurl }}/images/2015-12-13-1/US06996676-20060207-D00008.png" alt="ZFS-FUSE ARC US06996676 D00008">

In illustration A of FIG. 6, the list portions T1 405 and T2 415 together contain c pages and list portions B1 410 and B2 420 together contain c pages. In illustration B, a hit for page X is received in list portion B1 410. System 10 responds by increasing p, which increases the size of list portion T1 405 while decreasing the size of list portion T2 415. Window 425 effectively slides down. The distance window 425 moves in FIG. 6 is illustrative of the overall movement and is not based on actual values.

In the next illustration C of FIG. 6, one or more hits are received in list portion B2 420. System 10 responds by decreasing p, which decreases the size of list portion T1 405 while increasing the size of T2 415. Window 425 effectively slides up. Continuing with illustration D, another hit is received in list portion B2 420, so system 10 responds again by decreasing p and window 425 slides up again. If for example, a fourth hit is received in list portion B1 410, system 10 increases p, and window 425 slides down again as shown in illustration C. System 10 responds to the cache 15 workload, adjusting the sizes of list portions T1 405 and T2 415 to provide the maximum response to that workload.

One feature of the present system 10 is its resistance to scans, long streams of requests for pages not in cache 15. A page which is new to system 10, that is, not in L1 U L2, is placed in the MRU position of list L1 205 (block 538 and 554 of FIG. 5). From that position, the new page gradually makes its way to the LRU position in list L1 205. The new page does not affect list L2 210 before it is evicted, unless it is requested again. Consequently, a long stream of one-time-only reads will pass through list L1 205 without flushing out potentially important pages in list L2 210. In this case, system 10 is scan resistant in that it will only flush out pages in list portion T1 405 but not in list portion T2 415. Furthermore, when a scan begins, fewer hits will occur in list portion B1 410 than in list portion B2 420. Consequently, system 10 will continually decrease p, increasing list portion T2 415 at the expense of list portion T1 405. This will cause the one-time-only reads to pass through system 10 even faster, accentuating the scan resistance of system 10.

## Demostrating the Adjustable Replacement Cache (ARC)

The ARC solves the challenges either in it´s original implementation as well as in the extended implementation in ZFS. The following sections describe the fundamental concepts of the `Adaptice Replacement Cache`, as the additional mechanisms of the ZFS implementation hide a little bit the simplicity but effectiveness of this mechanism. Both implementations (the original `Adaptive Replacement Cache` and the ZFS `Adjustable Replacement Cache`) share their basic theory of operation thus the simplification is a viable one to explain ZFS ARC.

Let´s assume that you have a certain amount of pages in your cache. For simplicity, we will assume a 8 pages sized cache. For it´s operating the ARC needs a `directory table` as large as two times the size of of the cache.

This table seperates in 4 lists. The first two ones are obvious ones.

>- list for the most recently used pages
>- list for the most frequently used pages

The other two are a little bit stanger in their role. They are called `ghost lists`. They are filled with recently evicted pages from both lists:

>- list for the recently evicted pages from the list of most recently used pages
>- list of the recently eviced pages from the list of the most frequently used pages

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc1.png" alt="ZFS-FUSE ARC 1">

The both `ghost lists` doesn´t cache data, but a hit on them has an important effect to the behaviour of the cache as will be explained later on. So what happens in the cache? Let´s assume we read a page from the disk. We place it in the cache. The page is referenced in the recently used list.

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc2.png" alt="ZFS-FUSE ARC 2">

We read another different one. Its placed in the cache as well. And obviously it´s put into the recently used list at the most recently used position (position 1). 

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc3.png" alt="ZFS-FUSE ARC 3">

Okay, now we read the first page again. Now the reference is moved over to the frequently used list. It has to be used at least two times to get into this frequently used list. Whenever a page is accessed again on the frequently used page list, it is put to the beginning of the frequently used list again. By doing so really frequently accessed pages would stay in cache, but not too frequently accessed pages wanders to the end of the list and will get evicted at the end.

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc4.png" alt="ZFS-FUSE ARC 4">

Over the time both list fills and the cache is filled accordingly. But then one of the cache areas is full but you read an uncached page once again. One page has to be evicted from the cache to place a new one into it. Pages can just be evicted from cache, when the page in cache isn´t referenced by any of the non-ghost lists.

Let´s assume that we filled up the list for the recently used pages. 

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc5.png" alt="ZFS-FUSE ARC 5">

So it evicts the least recently used page from the cache. This page is put onto the list of recently evicted pages. 

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc6.png" alt="ZFS-FUSE ARC 6">

Now the page in the cache isn´t referenced any longer and thus you can evict it from the cache. The new page to cache gets now referenced by the cache directory.

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc7.png" alt="ZFS-FUSE ARC 7">

With every further eviction this page wanders to the end of the recently evicted list. At a later point in time the reference to this evicted page is at the end of the recently evicted list, and after the next eviction from the last recently used list it´s removed from this recently evicted list, and there is no reference in the lists anymore.

Okay, but what happens when we read a page that is on the list of already evicted pages again? Such an attempt to read leads to a phantom cache hit. As the data of the page has already been evicted from cache, the system has to read it from the media, but by this phantom cache hit, the system knows that this was a recently evicted page and not a page read just the first time or read a long time ago. The ARC can use this information to adjust itself to the load.

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc8.png" alt="ZFS-FUSE ARC 8">

Obviously this is a sign that our cache is too small. In this case the length of lists of the recently used pages in cache is increased by one. Obviously this reduces the place for frequently used pages by one.

<img src="{{ site.baseurl }}/images/2015-12-13-1/arc9.png" alt="ZFS-FUSE ARC 9">

But there is the same mechanism on the other side. If you get a hit on the list of recently evicted pages of the frequently used pages, it decreased the the list for currently cached recently used pages by one. This obviously increased the available space for frequently used pages by one.

With this behaviour the ARC adapts itself to the load. If the load is more like accessing recently accessed files, it would have more hits in the ghost list of the recently used files and thus increase the part of the cache for such files. And vice versa, if the load on the disks is more like accessing frequently accessed files, it would have more hits on the ghost lists for frequently accessed pages and thus increase the cache in favour of frequently used pages.

Furthermore this enables a neat feature: Let´s assume you read through a large amount of files for log file processing. You need every page only once. A normal least recently used based cache would load all the pages in the cache, thus evicting all frequently accessed pages. But as you access them only once they just fill up the cache without bringing you any advantages.

A cache based on ARC behaved differently. Obviously such a load would start to fill up the cache assigned for recently used pages really fast and such the pages are evicted soon. But as each of these pages are just accessed once, it´s highly unlikely that they will be hit in the ghost list for recently accessed files. Thus the part of the cache for recently accessed files doesn´t increase by such pages read only once. Let´s assume you you correlate the data in the logfiles with a large database (for simplification we assume, it doesn´t have an own caching mechanism). It accessed pages in the database files quite frequently. The probability of accessing pages referenced in the ghost list for frequently used files is much larger than on the ghost list for recently accessed files. Thus the cache for the frequently accessed database pages would increase. In essence, the cache would optimize itself for the database pages instead of polluting the cache with log files pages.

## The Modified Solaris ZFS ARC

As wrote before, the implementation in ZFS isn´t the pure ARC as described by both IBM researchers. It was extended in several ways:

* The ZFS ARC is variable in size and can react to the available memory. It can grow in size when memory is available or it can decrease in size when memory is needed for other things.
* The ZFS ARC can work with multiple block sized. The original implementation assumes an equal size for every block.
* You can lock pages in the cache to excempt them from eviction. This prevents the cache to evict pages, that are currently in use. The original implementation doesn´t have this feature, thus the algorithm to choose pages for eviction is lightly more complex in the ZFS ARC. It chooses the oldest evictable page for eviction.
* There are some other changes as can be exploited in the source code in `zfs-fuse/src/lib/libzpool/arc.c`.

The following code comments in `zfs-fuse/src/lib/libzpool/arc.c` can describe the ZFS ARC enhancments to the IMB ZRC in more details.

```c

        /*
         * DVA-based Adjustable Replacement Cache
         *
         * While much of the theory of operation used here is
         * based on the self-tuning, low overhead replacement cache
         * presented by Megiddo and Modha at FAST 2003, there are some
         * significant differences:
         *
         * 1. The Megiddo and Modha model assumes any page is evictable.
         * Pages in its cache cannot be "locked" into memory. This makes
         * the eviction algorithm simple: evict the last page in the list.
         * This also make the performance characteristics easy to reason
         * about. Our cache is not so simple. At any given moment, some
         * subset of the blocks in the cache are un-evictable because we
         * have handed out a reference to them. Blocks are only evictable
         * when there are no external references active. This makes
         * eviction far more problematic: We choose to evict the evictable
         * blocks that are the "lowest" in the list.
         *
         * There are times when it is not possible to evict the requested
         * space. In these circumstances we are unable to adjust the cache
         * size. To prevent the cache growing unbounded at these times we
         * implement a "cache throttle" that slows the flow of new data
         * into the cache until we can make space available.
         *
         * 2. The Megiddo and Modha model assumes a fixed cache size.
         * Pages are evicted when the cache is full and there is a cache
         * miss. Our model has a variable sized cache. It grows with
         * high use, but also tries to react to memory pressure from the
         * operating system: decreasing its size when system memory is
         * tight.
         *
         * 3. The Megiddo and Modha model assumes a fixed page size. All
         * elements of the cache are therefor exactly the same size. So
         * when adjusting the cache size following a cache miss, its simply
         * a matter of choosing a single page to evict. In our model, we
         * have variable sized cache blocks (rangeing from 512 bytes to
         * 128K bytes). We therefore choose a set of blocks to evict to make
         * space for a cache miss that approximates as closely as possible
         * the space used by the new block.
         *
         * See also:  "ARC: A Self-Tuning, Low Overhead Replacement Cache"
         * by N. Megiddo & D. Modha, FAST 2003
         */

```

## Level 2 ARC (L2ARC)

The L2ARC keeps the model stated in the paragraphs above untouched. The ARC doesn´t move the pages automatically to L2ARC instead of evictim them. Albeit this would seem the logical way, this would have severe impacts. At first a sequential read burst would overwrite large amounts of the L2ARC cache as such a burst would evict many pages in a short schedule. This is an unwanted behaviour.

The other problem: Let´s assume, your application needs a large heap of memory. The modified Solaris ZFS ARC is able to resize itself to the available memory. Thus when the applications request more memory, the ARC has to gets smaller. You have to evict a large amount of memory pages at once. If every page is written to L2ARC before eviction from ARC, this would add subtantial latency until your system can provide more memory, as you would have to wait for the L2ARC media before eviction.

The L2ARC mechanism tackles the problem a little bit differently: There is a thread called `l2arc_feed_thread` that walks over the soon to be evicted ends of both lists - the recently used list and the frequently used list - and puts them into a buffer (8 MB). From this place another thread (the `write_hand`) writes them into the L2ARC in a single write operation.

This algorithm has many advantages: The latency of freeing memory isn´t increased by the eviction. In situation of mass eviction by a sequential read burst, the blocks are evicted before the `l2arc_feed_thread` traverses the end of the the lists. So the effect of polluting the L2ARC by such read bursts is reduced (albeit not completly ruled out).

The following is the code comments for L2ARC from `zfs-fuse/src/lib/libzpool/arc.c`.

```c

        /*
         * Level 2 ARC
         *
         * The level 2 ARC (L2ARC) is a cache layer in-between main memory and disk.
         * It uses dedicated storage devices to hold cached data, which are populated
         * using large infrequent writes.  The main role of this cache is to boost
         * the performance of random read workloads.  The intended L2ARC devices
         * include short-stroked disks, solid state disks, and other media with
         * substantially faster read latency than disk.
         *
         *                 +-----------------------+
         *                 |         ARC           |
         *                 +-----------------------+
         *                    |         ^     ^
         *                    |         |     |
         *      l2arc_feed_thread()    arc_read()
         *                    |         |     |
         *                    |  l2arc read   |
         *                    V         |     |
         *               +---------------+    |
         *               |     L2ARC     |    |
         *               +---------------+    |
         *                   |    ^           |
         *          l2arc_write() |           |
         *                   |    |           |
         *                   V    |           |
         *                 +-------+      +-------+
         *                 | vdev  |      | vdev  |
         *                 | cache |      | cache |
         *                 +-------+      +-------+
         *                 +=========+     .-----.
         *                 :  L2ARC  :    |-_____-|
         *                 : devices :    | Disks |
         *                 +=========+    `-_____-'
         *
         * Read requests are satisfied from the following sources, in order:
         *
         *	1) ARC
         *	2) vdev cache of L2ARC devices
         *	3) L2ARC devices
         *	4) vdev cache of disks
         *	5) disks
         *
         * Some L2ARC device types exhibit extremely slow write performance.
         * To accommodate for this there are some significant differences between
         * the L2ARC and traditional cache design:
         *
         * 1. There is no eviction path from the ARC to the L2ARC.  Evictions from
         * the ARC behave as usual, freeing buffers and placing headers on ghost
         * lists.  The ARC does not send buffers to the L2ARC during eviction as
         * this would add inflated write latencies for all ARC memory pressure.
         *
         * 2. The L2ARC attempts to cache data from the ARC before it is evicted.
         * It does this by periodically scanning buffers from the eviction-end of
         * the MFU and MRU ARC lists, copying them to the L2ARC devices if they are
         * not already there.  It scans until a headroom of buffers is satisfied,
         * which itself is a buffer for ARC eviction.  The thread that does this is
         * l2arc_feed_thread(), illustrated below; example sizes are included to
         * provide a better sense of ratio than this diagram:
         *
         *	       head -->                        tail
         *	        +---------------------+----------+
         *	ARC_mfu |:::::#:::::::::::::::|o#o###o###|-->.   # already on L2ARC
         *	        +---------------------+----------+   |   o L2ARC eligible
         *	ARC_mru |:#:::::::::::::::::::|#o#ooo####|-->|   : ARC buffer
         *	        +---------------------+----------+   |
         *	             15.9 Gbytes      ^ 32 Mbytes    |
         *	                           headroom          |
         *	                                      l2arc_feed_thread()
         *	                                             |
         *	                 l2arc write hand <--[oooo]--'
         *	                         |           8 Mbyte
         *	                         |          write max
         *	                         V
         *		  +==============================+
         *	L2ARC dev |####|#|###|###|    |####| ... |
         *	          +==============================+
         *	                     32 Gbytes
         *
         * 3. If an ARC buffer is copied to the L2ARC but then hit instead of
         * evicted, then the L2ARC has cached a buffer much sooner than it probably
         * needed to, potentially wasting L2ARC device bandwidth and storage.  It is
         * safe to say that this is an uncommon case, since buffers at the end of
         * the ARC lists have moved there due to inactivity.
         *
         * 4. If the ARC evicts faster than the L2ARC can maintain a headroom,
         * then the L2ARC simply misses copying some buffers.  This serves as a
         * pressure valve to prevent heavy read workloads from both stalling the ARC
         * with waits and clogging the L2ARC with writes.  This also helps prevent
         * the potential for the L2ARC to churn if it attempts to cache content too
         * quickly, such as during backups of the entire pool.
         *
         * 5. After system boot and before the ARC has filled main memory, there are
         * no evictions from the ARC and so the tails of the ARC_mfu and ARC_mru
         * lists can remain mostly static.  Instead of searching from tail of these
         * lists as pictured, the l2arc_feed_thread() will search from the list heads
         * for eligible buffers, greatly increasing its chance of finding them.
         *
         * The L2ARC device write speed is also boosted during this time so that
         * the L2ARC warms up faster.  Since there have been no ARC evictions yet,
         * there are no L2ARC reads, and no fear of degrading read performance
         * through increased writes.
         *
         * 6. Writes to the L2ARC devices are grouped and sent in-sequence, so that
         * the vdev queue can aggregate them into larger and fewer writes.  Each
         * device is written to in a rotor fashion, sweeping writes through
         * available space then repeating.
         *
         * 7. The L2ARC does not store dirty content.  It never needs to flush
         * write buffers back to disk based storage.
         *
         * 8. If an ARC buffer is written (and dirtied) which also exists in the
         * L2ARC, the now stale L2ARC buffer is immediately dropped.
         *
         * The performance of the L2ARC can be tweaked by a number of tunables, which
         * may be necessary for different workloads:
         *
         *	l2arc_write_max		max write bytes per interval
         *	l2arc_write_boost	extra write bytes during device warmup
         *	l2arc_noprefetch	skip caching prefetched buffers
         *	l2arc_headroom		number of max device writes to precache
         *	l2arc_feed_secs		seconds between L2ARC writing
         *
         * Tunables may be removed or added as future performance improvements are
         * integrated, and also may become zpool properties.
         *
         * There are three key functions that control how the L2ARC warms up:
         *
         *	l2arc_write_eligible()	check if a buffer is eligible to cache
         *	l2arc_write_size()	calculate how much to write
         *	l2arc_write_interval()	calculate sleep delay between writes
         *
         * These three functions determine what to write, how much, and how quickly
         * to send writes.
         */

```

## Conclusion

The design of Adjustable Replacement Cache is much more efficient than ordinary LRU cache design. *Megiddo* and *Modha* were able to show much better hit rates with their original `Adaptive Replacement Cache`. The ZFS ARC shares the basic theory of operation thus the hit rate advantages should be similar to the one of the original design. More important: The idea of using large caches in form of SSD gets even more viable, if the cache algorithm helps them to yield an even better hit rate.

## References

This blog entry is mostly an edit based on the following sources, credits should go to these authors!

* https://claudidays.files.wordpress.com/2011/06/zfs_arc.pdf
* http://www.c0t0d0s0.org/archives/5329-Some-insight-into-the-read-cache-of-ZFS-or-The-ARC.html
* http://wiki.illumos.org/plugins/viewsource/viewpagesrc.action?pageId=1148612
* https://en.wikipedia.org/wiki/Adaptive_replacement_cache
* http://www.google.com.hk/patents/US6996676