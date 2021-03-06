---
layout: post
title: "ZFS-FUSE ARC Cache Code Analysis"
description: 
headline: 
modified: 2015-12-14
category: FileSysDev
tags: [Filesystem]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

The ARC achieves a high cache hit rate by using multiple cache algorithms at the same time: most recently used (MRU) and most frequently used (MFU). Main memory is balanced between these algorithms based on their performance, which is known by maintaining extra metadata (in main memory) to see how each algorithm would perform if it ruled all of memory. Such extra metadata is held on `ghost lists`. The MRU + MFU lists refer to the data cached in main memory; the MRU ghost + MFU ghost lists consist of themselves only (the metadata) to track algorithm performance. The current version of the ZFS ARC splits the lists into separate data and metadata lists, and also has a list for anonymous buffers and one for L2ARC only buffers (which I added when I developed the L2ARC). This blog post tries to analyze the ZFS ARC from source code perspective.

## ARC Data Structures

Data exists in the ARC cache as buffers. The data structures used to maintain the ARC are `arc_buf_hdr_t` and `arc_buf_t`. These data structures are used to determine if a buffer is in ARC, and, if so, where it is (`mru`, `mfu`, `mru ghost`, `mfu ghost`, `l2arc`). (The `ghost`b lists are used to determine when a `mru` or `mfu` cache is too small). But they do not identify what object the data/metadata holds. For this, the `dmu_buf_impl_t` structure (hereafter referred to as "`dbuf`" structures) can be used. Note that not everything in the ARC is mapped by `dbufs`.

The `arc_buf_hdr_t` is defined to be `struct arc_buf_hdr` in `zfs-fuse/src/lib/libzpool/arc.c` as below:

```c

	struct arc_buf_hdr {
		/* protected by hash lock */
		dva_t			b_dva;
		uint64_t		b_birth;
		uint64_t		b_cksum0;
	
		kmutex_t		b_freeze_lock;
		zio_cksum_t		*b_freeze_cksum;
	
		arc_buf_hdr_t		*b_hash_next;
		arc_buf_t		*b_buf;
		uint32_t		b_flags;
		uint32_t		b_datacnt;
	
		arc_callback_t		*b_acb;
		kcondvar_t		b_cv;
	
		/* immutable */
		arc_buf_contents_t	b_type;
		uint64_t		b_size;
		uint64_t		b_spa;
	
		/* protected by arc state mutex */
		arc_state_t		*b_state;
		list_node_t		b_arc_node;
	
		/* updated atomically */
		clock_t			b_arc_access;
	
		/* self protecting */
		refcount_t		b_refcnt;
	
		l2arc_buf_hdr_t		*b_l2hdr;
		list_node_t		b_l2node;
	};

```

* In the code, `dva_t` is meant for 128-bit ZFS `Data Virtual Address (DVA)`, which is further defined to be the following structure in `zfs-fuse/src/lib/libzfscommon/include/sys/spa.h`:

```c

	/*
	 * All SPA data is represented by 128-bit data virtual addresses (DVAs).
	 * The members of the dva_t should be considered opaque outside the SPA.
	 */
	typedef struct dva {
		uint64_t	dva_word[2];
	} dva_t;

```

* When you see `spa_t`, it means `Storage Pool Allocator (SPA)`, which is defined to be a structure (not to be detailed here). But when you see something like `uint64_t spa`, it means the `guid` of the `spa_t` (obtained by `guid = spa_guid(spa)`).

* The `b_birth` filed in `arc_buf_hdr_t` is the filled by code such as `hdr->b_birth = BP_PHYSICAL_BIRTH(bp)` when the buffer header enters the cache, either during `arc_read_nolock()` or `arc_write_done()` completes the actual IO to read or write data. 

* The `b_buf` is actually starting a list of `struct arc_buf`. That is, `arc_buf_hdr_t` is actually the holding data structure of a list of buffers represented by `struct arc_buf`. Each `struct arc_buf` in that list has a back pinter `b_hdr` that points to the holding `arc_buf_hdr_t`.

```c

	struct arc_buf {
		arc_buf_hdr_t		*b_hdr;
		arc_buf_t		*b_next;
		krwlock_t		b_lock;
		void			*b_data;
		arc_evict_func_t	*b_efunc;
		void			*b_private;
	};

```

* Access to these data structures is protected by a hash table `buf_hash_table` based on the 128-bit ZFS `data virtual address (DVA)`. The hash table has 256 buffer locks (`BUF_LOCKS`), each is a padded lock `ht_lock` (to avoid false sharing). The `ht_table` is initialized in `buf_init()` to be an array of pointers which can point to all available physical memory blocks (each block is 64KB in size). According to the comments in `buf_init()`, each 1GB of physical memory would require 128KB of `ht_table` memory. Note the `buf_hash_table` is a global variable as a single instance of `struct buf_hash_table`. 

```c

	/*
	 * Hash table routines
	 */
	
	#define	HT_LOCK_PAD	64
	
	struct ht_lock {
		kmutex_t	ht_lock;
	#ifdef _KERNEL
		unsigned char	pad[(HT_LOCK_PAD - sizeof (kmutex_t))];
	#endif
	};
	
	#define	BUF_LOCKS 256
	typedef struct buf_hash_table {
		uint64_t ht_mask;
		arc_buf_hdr_t **ht_table;
		struct ht_lock ht_locks[BUF_LOCKS];
	} buf_hash_table_t;
	
	static buf_hash_table_t buf_hash_table;

```

* The `b_state` is the pointer to a `arc_state_t` that is one of the variables in `ARC_anon`, `ARC_mru`, `ARC_mru_ghost`, `ARC_mfu`, `ARC_mfu_ghost`, and `ARC_l2c_only`. Each of `arc_state_t` actually contains two lists in `arcs_list[]`, indexed by `ARC_BUFC_DATA` or `ARC_BUFC_METADATA`, representing data or meta buffers of these different cache states. The `arc_buf_hdr_t` instances are added to the list when a buffer changes state, by calling `arc_change_state()`.

```c
	
	typedef enum arc_buf_contents {
		ARC_BUFC_DATA,				/* buffer contains data */
		ARC_BUFC_METADATA,			/* buffer contains metadata */
		ARC_BUFC_NUMTYPES
	} arc_buf_contents_t;

	typedef struct arc_state {
	        list_t  arcs_list[ARC_BUFC_NUMTYPES];   /* list of evictable buffers */
	        uint64_t arcs_lsize[ARC_BUFC_NUMTYPES]; /* amount of evictable data */
	        uint64_t arcs_size;     /* total amount of data in this state */
	        kmutex_t arcs_mtx;
	} arc_state_t;
	
	/* The 6 states: */
	static arc_state_t ARC_anon;
	static arc_state_t ARC_mru;
	static arc_state_t ARC_mru_ghost;
	static arc_state_t ARC_mfu;
	static arc_state_t ARC_mfu_ghost;
	static arc_state_t ARC_l2c_only;

	static arc_state_t	*arc_anon;
	static arc_state_t	*arc_mru;
	static arc_state_t	*arc_mru_ghost;
	static arc_state_t	*arc_mfu;
	static arc_state_t	*arc_mfu_ghost;
	static arc_state_t	*arc_l2c_only;

```

## ARC Initialization

The ARC Initialization is performed by calling `arc_init()`. It initializes the basic control variables, buffers, lists and locks, sets up stats reporting infrastructure, and starts an ARC Reclaim Thread. The code is listed below and we will talk about the various initialization operations after the code.

```c

	void arc_init(void)
	{
		mutex_init(&arc_reclaim_thr_lock, NULL, MUTEX_DEFAULT, NULL);
		cv_init(&arc_reclaim_thr_cv, NULL, CV_DEFAULT, NULL);
	
		/* Convert seconds to clock ticks */
		arc_min_prefetch_lifespan = 1 * hz;
	
		/* Start out with 1/8 of all memory */
		arc_c = physmem * PAGESIZE / 8;
	
	#if 0
		/*
		 * On architectures where the physical memory can be larger
		 * than the addressable space (intel in 32-bit mode), we may
		 * need to limit the cache to 1/8 of VM size.
		 */
		arc_c = MIN(arc_c, vmem_size(heap_arena, VMEM_ALLOC | VMEM_FREE) / 8);
	#endif
	
		/* set min cache to 16 MB */
		arc_c_min = 16<<20;
		if (max_arc_size) {
			if (max_arc_size < arc_c_min) {
				syslog(LOG_WARNING,"max_arc_size too small (" FI64 " bytes), using arc_c_min (" FI64 " bytes)",max_arc_size,arc_c_min);
				arc_c_max = arc_c_min;
			} else {
				arc_c_max = max_arc_size;
			}
		} else {
	#ifdef _KERNEL
		/* set max cache to ZFSFUSE_MAX_ARCSIZE */
		arc_c_max = ZFSFUSE_MAX_ARCSIZE;
	#else
		arc_c_max = 64<<20;
	#endif
		}
		syslog(LOG_NOTICE,"ARC setup: min ARC size set to " FI64 " bytes",arc_c_min);
		syslog(LOG_NOTICE,"ARC setup: max ARC size set to " FI64 " bytes",arc_c_max);
	
		/*
		 * Allow the tunables to override our calculations if they are
		 * reasonable (ie. over 64MB)
		 */
		if (zfs_arc_max > 64<<20 && zfs_arc_max < physmem * PAGESIZE)
			arc_c_max = zfs_arc_max;
		if (zfs_arc_min > 64<<20 && zfs_arc_min <= arc_c_max)
			arc_c_min = zfs_arc_min;
	
		arc_c = arc_c_max;
		arc_p = (arc_c >> 1);
	
		/* limit meta-data to 1/4 of the arc capacity */
		arc_meta_limit = arc_c_max / 4;
	
		/* Allow the tunable to override if it is reasonable */
		if (zfs_arc_meta_limit > 0 && zfs_arc_meta_limit <= arc_c_max)
			arc_meta_limit = zfs_arc_meta_limit;
	
		if (arc_c_min < arc_meta_limit / 2 && zfs_arc_min == 0)
			arc_c_min = arc_meta_limit / 2;
	
		if (zfs_arc_grow_retry > 0)
			arc_grow_retry = zfs_arc_grow_retry;
	
		if (zfs_arc_shrink_shift > 0)
			arc_shrink_shift = zfs_arc_shrink_shift;
	
		if (zfs_arc_p_min_shift > 0)
			arc_p_min_shift = zfs_arc_p_min_shift;
	
		/* if kmem_flags are set, lets try to use less memory */
		if (kmem_debugging())
			arc_c = arc_c / 2;
		if (arc_c < arc_c_min)
			arc_c = arc_c_min;
	
		arc_anon = &ARC_anon;
		arc_mru = &ARC_mru;
		arc_mru_ghost = &ARC_mru_ghost;
		arc_mfu = &ARC_mfu;
		arc_mfu_ghost = &ARC_mfu_ghost;
		arc_l2c_only = &ARC_l2c_only;
		arc_size = 0;
	
		mutex_init(&arc_anon->arcs_mtx, NULL, MUTEX_DEFAULT, NULL);
		mutex_init(&arc_mru->arcs_mtx, NULL, MUTEX_DEFAULT, NULL);
		mutex_init(&arc_mru_ghost->arcs_mtx, NULL, MUTEX_DEFAULT, NULL);
		mutex_init(&arc_mfu->arcs_mtx, NULL, MUTEX_DEFAULT, NULL);
		mutex_init(&arc_mfu_ghost->arcs_mtx, NULL, MUTEX_DEFAULT, NULL);
		mutex_init(&arc_l2c_only->arcs_mtx, NULL, MUTEX_DEFAULT, NULL);
	
		list_create(&arc_mru->arcs_list[ARC_BUFC_METADATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_mru->arcs_list[ARC_BUFC_DATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_mru_ghost->arcs_list[ARC_BUFC_METADATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_mru_ghost->arcs_list[ARC_BUFC_DATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_mfu->arcs_list[ARC_BUFC_METADATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_mfu->arcs_list[ARC_BUFC_DATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_mfu_ghost->arcs_list[ARC_BUFC_METADATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_mfu_ghost->arcs_list[ARC_BUFC_DATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_l2c_only->arcs_list[ARC_BUFC_METADATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
		list_create(&arc_l2c_only->arcs_list[ARC_BUFC_DATA],
		    sizeof (arc_buf_hdr_t), offsetof(arc_buf_hdr_t, b_arc_node));
	
		buf_init();
	
		arc_thread_exit = 0;
		arc_eviction_list = NULL;
		mutex_init(&arc_eviction_mtx, NULL, MUTEX_DEFAULT, NULL);
		bzero(&arc_eviction_hdr, sizeof (arc_buf_hdr_t));
	
		arc_ksp = kstat_create("zfs", 0, "arcstats", "misc", KSTAT_TYPE_NAMED,
		    sizeof (arc_stats) / sizeof (kstat_named_t), KSTAT_FLAG_VIRTUAL);
	
		if (arc_ksp != NULL) {
			arc_ksp->ks_data = &arc_stats;
			kstat_install(arc_ksp);
		}
	
		(void) thread_create(NULL, 0, arc_reclaim_thread, NULL, 0, &p0,
		    TS_RUN, minclsyspri);
	
		arc_dead = FALSE;
		arc_warm = B_FALSE;
	
		if (zfs_write_limit_max == 0)
			zfs_write_limit_max = ptob(physmem) >> zfs_write_limit_shift;
		else
			zfs_write_limit_shift = 0;
		mutex_init(&zfs_write_limit_lock, NULL, MUTEX_DEFAULT, NULL);
	}

```

* Initialize `arc_reclaim_thr_lock` and `arc_reclaim_thr_cv`, so that the `arc_reclaim_thread()` thread can be synchronized by the mutex and condition variable. The ARC reclaim thread `arc_reclaim_thread()`, which wakes up every second (or sooner if signaled by the `arc_reclaim_thr_cv` conditional variable) and will attempt to reduce the size of the ARC to the target size. See **ARC Reclaim Thread** for more details.
* Setup `arc_min_prefetch_lifespan` to be 1 second (in ticks), used as the minimum lifespan of a prefetch block. In `arc_evict()` a check `lbolt - ab->b_arc_access < arc_min_prefetch_lifespan` is made for buffers that has `(ARC_PREFETCH|ARC_INDIRECT)` flags set, this is to make sure a prefetch block to be living in the cache for a minimum of time in order to be effective (shorter to live makes the prefetch operation cost rate higher).
* Initialize the `arc_c` variable as the `Target Size` of ARC cache, initially to start out with 1/8 of all memory. The value of `arc_c` is in bytes (while `physmem` is in pages as retrived by `physmem = sysconf(_SC_PHYS_PAGES)` in `kernel_init()` of `zfs-fuse/src/lib/libzpool/kernel.c`). This value of `arc_c` is covered in details in the previous blog post **ZFS-FUSE ARC CACHE THEORY**. Note that the "Target Size" `arc_c` is a constantly changing number, like a stock price, of what the ARC thinks it should be, while the "Current Size" `arc_size` is how large the ARC really is.
* Setup `arc_c_min` to 16MB, and `arc_c_max` according to tunables `max_arc_size` and `ZFSFUSE_MAX_ARCSIZE` configuration parameter. The `arc_c` variable is finally updated to be `arc_c_max`.
* Setup `arc_p` (`Most Recently Used Cache Size`) to be half of `arc_c`. This variable indicates how the ARC is adapting its MRU and MFU list size depending on the workload. The `Most Frequently Used Cache Size` would be `arc_c - arc_p`. This value of `arc_p` is covered in details in the previous blog post **ZFS-FUSE ARC CACHE THEORY**. Note that this `arc_p` is vital for ARC performance and it is tunable by the user. It should be better that the `REAL Hit Ratio` of the ARC Cache `MRU/MFU = arc_p/(arc_c - arc_p)` be kept during `arc_shrink()`. See [this link](https://github.com/zfsonlinux/zfs/issues/2167) and [this link](http://www.cuddletech.com/blog/pivot/entry.php?id=979) for a discussion. This might be something to optimize for ZFS-FUSE. Note that ZFS considers an `Anon buffer (New Customer, First Cache Hit)` as a hit... but its not really, so it should be removed for the "REAL" ratio.
* Some other tunables such as `zfs_arc_meta_limit`, `zfs_arc_grow_retry`, `zfs_arc_shrink_shift`, `zfs_arc_p_min_shift` are checked to initialize some similarly named ARC internal variables like `arc_meta_limit`, `arc_grow_retry`, etc.
* Several cache list pointers of `arc_state_t` are initilized. Note that each of these cache list has a `arcs_mtx` mutex to be initialized by `mutex_init()`. Also, each list actually has two lists for different types of data, `arcs_list[ARC_BUFC_METADATA]` and `arcs_list[ARC_BUFC_DATA]`, these lists are created by `list_create()`. Strangely, `arc_anon` does not have these listed created. This might need to be checked.

  >- Anon (`arc_anon = &ARC_anon`) [New Customer, First Cache Hit]
  >- Most Recently Used (`arc_mru = &ARC_mru`) [Return Customer]
  >- Most Frequently Used (`arc_mfu = &ARC_mfu`) [Frequent Customer]
  >- Most Recently Used Ghost (`arc_mru_ghost = &ARC_mru_ghost`) [Return Customer Evicted, Now Back]
  >- Most Frequently Used Ghost (`arc_mfu_ghost = &ARC_mfu_ghost`) [Frequent Customer Evicted, Now Back]
  >- L2ARC Only (`arc_l2c_only = &ARC_l2c_only`) [Exists in L2ARC but Not Others]

* Call `buf_init()` to initialize the ARC buffers. See **ARC Buffer Initialization** for details.
* Initialize `arc_thread_exit`, `arc_eviction_list`, `arc_eviction_mtx`, and `arc_eviction_hdr` which are used to keep track of ARC Eviction.
* Calling `kstat_create()` to create `arcstats`.
* Create ARC Reclaim Thread running in `arc_reclaim_thread()` by calling `thread_create()`.
* Initialize `zfs_write_limit_shift` based on `zfs_write_limit_max`. 

The following is the detailed flowchart.

<img src="{{ site.baseurl }}/images/2015-12-14-1/ControlFlowGraph-arc_init.png" alt="ZFS-FUSE Control Flow Graph arc_init">

## ARC Buffer Initialization

```c

	static void buf_init(void)
	{
		uint64_t *ct;
		uint64_t hsize = 1ULL << 12;
		int i, j;
	
		/*
		 * The hash table is big enough to fill all of physical memory
		 * with an average 64K block size.  The table will take up
		 * totalmem*sizeof(void*)/64K (eg. 128KB/GB with 8-byte pointers).
		 */
		while (hsize * 65536 < physmem * PAGESIZE)
			hsize <<= 1;
	retry:
		buf_hash_table.ht_mask = hsize - 1;
		buf_hash_table.ht_table =
		    kmem_zalloc(hsize * sizeof (void*), KM_NOSLEEP);
		if (buf_hash_table.ht_table == NULL) {
			ASSERT(hsize > (1ULL << 8));
			hsize >>= 1;
			goto retry;
		}
	
		hdr_cache = kmem_cache_create("arc_buf_hdr_t", sizeof (arc_buf_hdr_t),
		    0, hdr_cons, hdr_dest, hdr_recl, NULL, NULL, 0);
		buf_cache = kmem_cache_create("arc_buf_t", sizeof (arc_buf_t),
		    0, buf_cons, buf_dest, NULL, NULL, NULL, 0);
	
		for (i = 0; i < 256; i++)
			for (ct = zfs_crc64_table + i, *ct = i, j = 8; j > 0; j--)
				*ct = (*ct >> 1) ^ (-(*ct & 1) & ZFS_CRC64_POLY);
	
		for (i = 0; i < BUF_LOCKS; i++) {
			mutex_init(&buf_hash_table.ht_locks[i].ht_lock,
			    NULL, MUTEX_DEFAULT, NULL);
		}
	}

```

## ARC Exit

The ARC is removed by calling `arc_fini()`, which tears down the ARC Reclaim Thread and destroy the various buffers, lists, and locks.

```c

	void arc_fini(void)
	{
		mutex_enter(&arc_reclaim_thr_lock);
		arc_thread_exit = 1;
		while (arc_thread_exit != 0)
			cv_wait(&arc_reclaim_thr_cv, &arc_reclaim_thr_lock);
		mutex_exit(&arc_reclaim_thr_lock);
	
		arc_flush(NULL);
	
		arc_dead = TRUE;
	
		if (arc_ksp != NULL) {
			kstat_delete(arc_ksp);
			arc_ksp = NULL;
		}
	
		mutex_destroy(&arc_eviction_mtx);
		mutex_destroy(&arc_reclaim_thr_lock);
		cv_destroy(&arc_reclaim_thr_cv);
	
		list_destroy(&arc_mru->arcs_list[ARC_BUFC_METADATA]);
		list_destroy(&arc_mru_ghost->arcs_list[ARC_BUFC_METADATA]);
		list_destroy(&arc_mfu->arcs_list[ARC_BUFC_METADATA]);
		list_destroy(&arc_mfu_ghost->arcs_list[ARC_BUFC_METADATA]);
		list_destroy(&arc_mru->arcs_list[ARC_BUFC_DATA]);
		list_destroy(&arc_mru_ghost->arcs_list[ARC_BUFC_DATA]);
		list_destroy(&arc_mfu->arcs_list[ARC_BUFC_DATA]);
		list_destroy(&arc_mfu_ghost->arcs_list[ARC_BUFC_DATA]);
	
		mutex_destroy(&arc_anon->arcs_mtx);
		mutex_destroy(&arc_mru->arcs_mtx);
		mutex_destroy(&arc_mru_ghost->arcs_mtx);
		mutex_destroy(&arc_mfu->arcs_mtx);
		mutex_destroy(&arc_mfu_ghost->arcs_mtx);
		mutex_destroy(&arc_l2c_only->arcs_mtx);
	
		mutex_destroy(&zfs_write_limit_lock);
	
		buf_fini();
	
		ASSERT(arc_loaned_bytes == 0);
	}

```

The following is the detailed flowchart.

<img src="{{ site.baseurl }}/images/2015-12-14-1/ControlFlowGraph-arc_fini.png" alt="ZFS-FUSE Control Flow Graph arc_fini">

## ARC Reclaim Thread

The ARC reclaim thread is implemented in `arc_reclaim_thread()`, which wakes up every second (or sooner if signaled by the `arc_reclaim_thr_cv` conditional variable) and will attempt to reduce the size of the ARC to the "Target Size". It calls `arc_kmem_reap_now()` to clean up the kmem caches, and `arc_adjust()` to resize the ARC lists. If `arc_shrink()` is called by `arc_kmem_reap_now()`, the target ARC size is reduced by `arc_shrink_shift` (or `needfree`), which means shrinking the ARC by 3%. If you plot the ARC size, you sometimes see these `arc_shrink()` steps appearing as teeth on a saw – a sharp drop followed by a gradual increase.

```c

	static void arc_reclaim_thread(void)
	{
		uint64_t			growtime = 0;
		arc_reclaim_strategy_t	last_reclaim = ARC_RECLAIM_CONS;
		callb_cpr_t		cpr;
	
		CALLB_CPR_INIT(&cpr, &arc_reclaim_thr_lock, callb_generic_cpr, FTAG);
	
		mutex_enter(&arc_reclaim_thr_lock);
		while (arc_thread_exit == 0) {
			if (arc_reclaim_needed()) {
	
				if (arc_no_grow) {
					if (last_reclaim == ARC_RECLAIM_CONS) {
						last_reclaim = ARC_RECLAIM_AGGR;
					} else {
						last_reclaim = ARC_RECLAIM_CONS;
					}
				} else {
					arc_no_grow = TRUE;
					last_reclaim = ARC_RECLAIM_AGGR;
					membar_producer();
				}
	
				/* reset the growth delay for every reclaim */
				growtime = lbolt64 + (arc_grow_retry * hz);
	
				arc_kmem_reap_now(last_reclaim);
				arc_warm = B_TRUE;
	
			} else if (arc_no_grow && lbolt64 >= growtime) {
				arc_no_grow = FALSE;
			}
	
			if (2 * arc_c < arc_size +
			    arc_mru_ghost->arcs_size + arc_mfu_ghost->arcs_size)
				arc_adjust();
	
			if (arc_eviction_list != NULL)
				arc_do_user_evicts();
	
			/* block until needed, or one second, whichever is shorter */
			CALLB_CPR_SAFE_BEGIN(&cpr);
			(void) cv_timedwait(&arc_reclaim_thr_cv,
			    &arc_reclaim_thr_lock, (lbolt + hz));
			CALLB_CPR_SAFE_END(&cpr, &arc_reclaim_thr_lock);
		}
	
		arc_thread_exit = 0;
		cv_broadcast(&arc_reclaim_thr_cv);
		CALLB_CPR_EXIT(&cpr);		/* drops arc_reclaim_thr_lock */
		thread_exit();
	}

```

ZFS ARC growing more than the "Target Size" is allowed, but `arc_reclaim_thread()` must eventually wake up to reduce the actual size to the "Target Size". Note that ZFS-FUSE currently disabled `arc_reclaim_needed()` by just returning 0, which makes `arc_reclaim_thread()` to force `arc_no_grow = FALSE` in the `else` condition. Then the follow on actions by the loop is just two things:

1) If "actual total arc size" (`arc_size`) plus "glost list arc size" (`arc_mru_ghost->arcs_size + arc_mfu_ghost->arcs_size`) is larger than two times of "Target Size" (`arc_c`), then `arc_adjust()` is called to adjust (ie evict) cache contents to new sizes.

2) If `arc_eviction_list` is not NULL, call `arc_do_user_evicts()` to remove the ARC buffers. Arc buffers may have an associated eviction callback function. This function will be invoked prior to removing the buffer (e.g. in `arc_do_user_evicts()`).  Note however that the data associated with the buffer may be evicted prior to the callback.  The callback must be made with *no locks held* (to prevent deadlock).  Additionally, the users of callbacks must ensure that their private data is protected from simultaneous callbacks from `arc_buf_evict()` and `arc_do_user_evicts()`.

The following is the detailed flowchart for `arc_reclaim_thread()` function.

<img src="{{ site.baseurl }}/images/2015-12-14-1/ControlFlowGraph-arc_reclaim_thread.png" alt="ZFS-FUSE Control Flow Graph arc_reclaim_thread">

### `arc_adjust` 

```c

	static void arc_adjust(void)
	{
		int64_t adjustment, delta;
	
		/*
		 * Adjust MRU size
		 */
	
		adjustment = MIN(arc_size - arc_c,
		    arc_anon->arcs_size + arc_mru->arcs_size + arc_meta_used - arc_p);
	
		if (adjustment > 0 && arc_mru->arcs_lsize[ARC_BUFC_DATA] > 0) {
			delta = MIN(arc_mru->arcs_lsize[ARC_BUFC_DATA], adjustment);
			(void) arc_evict(arc_mru, 0, delta, FALSE, ARC_BUFC_DATA);
			adjustment -= delta;
		}
	
		if (adjustment > 0 && arc_mru->arcs_lsize[ARC_BUFC_METADATA] > 0) {
			delta = MIN(arc_mru->arcs_lsize[ARC_BUFC_METADATA], adjustment);
			(void) arc_evict(arc_mru, 0, delta, FALSE,
			    ARC_BUFC_METADATA);
		}
	
		/*
		 * Adjust MFU size
		 */
	
		adjustment = arc_size - arc_c;
	
		if (adjustment > 0 && arc_mfu->arcs_lsize[ARC_BUFC_DATA] > 0) {
			delta = MIN(adjustment, arc_mfu->arcs_lsize[ARC_BUFC_DATA]);
			(void) arc_evict(arc_mfu, 0, delta, FALSE, ARC_BUFC_DATA);
			adjustment -= delta;
		}
	
		if (adjustment > 0 && arc_mfu->arcs_lsize[ARC_BUFC_METADATA] > 0) {
			int64_t delta = MIN(adjustment,
			    arc_mfu->arcs_lsize[ARC_BUFC_METADATA]);
			(void) arc_evict(arc_mfu, 0, delta, FALSE,
			    ARC_BUFC_METADATA);
		}
	
		/*
		 * Adjust ghost lists
		 */
	
		adjustment = arc_mru->arcs_size + arc_mru_ghost->arcs_size - arc_c;
	
		if (adjustment > 0 && arc_mru_ghost->arcs_size > 0) {
			delta = MIN(arc_mru_ghost->arcs_size, adjustment);
			arc_evict_ghost(arc_mru_ghost, 0, delta);
		}
	
		adjustment =
		    arc_mru_ghost->arcs_size + arc_mfu_ghost->arcs_size - arc_c;
	
		if (adjustment > 0 && arc_mfu_ghost->arcs_size > 0) {
			delta = MIN(arc_mfu_ghost->arcs_size, adjustment);
			arc_evict_ghost(arc_mfu_ghost, 0, delta);
		}
	}	

```

### arc_shrink() reduces the ARC Target Size

```c

        void arc_shrink(void)
        {
        	if (arc_c > arc_c_min) {
        		uint64_t to_free;
        
        #if 0
        		to_free = MAX(arc_c >> arc_shrink_shift, ptob(needfree));
        #else
        		to_free = arc_c >> arc_shrink_shift;
        #endif
        		if (arc_c > arc_c_min + to_free)
        			atomic_add_64(&arc_c, -to_free);
        		else
        			arc_c = arc_c_min;
        
        		atomic_add_64(&arc_p, -(arc_p >> arc_shrink_shift));
        		if (arc_c > arc_size)
        			arc_c = MAX(arc_size, arc_c_min);
        		if (arc_p > arc_c)
        			arc_p = (arc_c >> 1);
        		ASSERT(arc_c >= arc_c_min);
        		ASSERT((int64_t)arc_p >= 0);
        	}
        
        	if (arc_size > arc_c)
        		arc_adjust();
        }

```

The `arc_shrink()` function reduces the arc target size by doing the following:

* Firstly, `to_free` is calculated as follow: `to_free = arc_c >> arc_shrink_shift`.
* Then it will guarantee that `to_free` will not reduce `target_size` to a value lower than the minimum target size.
 
>- If the condition above is true, then "Target Size" `arc_c` is reduced by `to_free` (which means reducing the arc size by about 3.125%).
>- If not, "Target Size" `arc_c` is set to the minimum target size `arc_c_min`.

* And finally, if "Target Size" `arc_c` is smaller than the "Current Size" `arc_size`, `arc_adjust()` is called to do the actual work.


The following is the detailed flowchart for `arc_shrink()` function:

<img src="{{ site.baseurl }}/images/2015-12-14-1/ControlFlowGraph-arc_shrink.png" alt="ZFS-FUSE Control Flow Graph arc_shrink">

## ARC Eviction

```c

	/*
	 * Evict buffers from list until we've removed the specified number of
	 * bytes.  Move the removed buffers to the appropriate evict state.
	 * If the recycle flag is set, then attempt to "recycle" a buffer:
	 * - look for a buffer to evict that is `bytes' long.
	 * - return the data block from this buffer rather than freeing it.
	 * This flag is used by callers that are trying to make space for a
	 * new buffer in a full arc cache.
	 *
	 * This function makes a "best effort".  It skips over any buffers
	 * it can't get a hash_lock on, and so may not catch all candidates.
	 * It may also return without evicting as much space as requested.
	 */
	static void * arc_evict(arc_state_t *state, uint64_t spa, int64_t bytes, boolean_t recycle,
	    arc_buf_contents_t type)
	{
		arc_state_t *evicted_state;
		uint64_t bytes_evicted = 0, skipped = 0, missed = 0;
		arc_buf_hdr_t *ab, *ab_prev = NULL;
		list_t *list = &state->arcs_list[type];
		kmutex_t *hash_lock;
		boolean_t have_lock;
		void *stolen = NULL;
	
		ASSERT(state == arc_mru || state == arc_mfu);
	
		evicted_state = (state == arc_mru) ? arc_mru_ghost : arc_mfu_ghost;
	
		mutex_enter(&state->arcs_mtx);
		mutex_enter(&evicted_state->arcs_mtx);
	
		for (ab = list_tail(list); ab; ab = ab_prev) {
			ab_prev = list_prev(list, ab);
			/* prefetch buffers have a minimum lifespan */
			if (HDR_IO_IN_PROGRESS(ab) ||
			    (spa && ab->b_spa != spa) ||
			    (ab->b_flags & (ARC_PREFETCH|ARC_INDIRECT) &&
			    lbolt - ab->b_arc_access < arc_min_prefetch_lifespan)) {
				skipped++;
				continue;
			}
			/* "lookahead" for better eviction candidate */
			if (recycle && ab->b_size != bytes &&
			    ab_prev && ab_prev->b_size == bytes)
				continue;
			hash_lock = HDR_LOCK(ab);
			have_lock = MUTEX_HELD(hash_lock);
			if (have_lock || mutex_tryenter(hash_lock)) {
				ASSERT3U(refcount_count(&ab->b_refcnt), ==, 0);
				ASSERT(ab->b_datacnt > 0);
				while (ab->b_buf) {
					arc_buf_t *buf = ab->b_buf;
					if (!rw_tryenter(&buf->b_lock, RW_WRITER)) {
						missed += 1;
						break;
					}
					if (buf->b_data) {
						bytes_evicted += ab->b_size;
						if (recycle && ab->b_type == type &&
						    ab->b_size == bytes &&
						    !HDR_L2_WRITING(ab)) {
							stolen = buf->b_data;
							recycle = FALSE;
						}
					}
					if (buf->b_efunc) {
						mutex_enter(&arc_eviction_mtx);
						arc_buf_destroy(buf,
						    buf->b_data == stolen, FALSE);
						ab->b_buf = buf->b_next;
						buf->b_hdr = &arc_eviction_hdr;
						buf->b_next = arc_eviction_list;
						arc_eviction_list = buf;
						mutex_exit(&arc_eviction_mtx);
						rw_exit(&buf->b_lock);
					} else {
						rw_exit(&buf->b_lock);
						arc_buf_destroy(buf,
						    buf->b_data == stolen, TRUE);
					}
				}
	
				if (ab->b_l2hdr) {
					ARCSTAT_INCR(arcstat_evict_l2_cached,
					    ab->b_size);
				} else {
					if (l2arc_write_eligible(ab->b_spa, ab)) {
						ARCSTAT_INCR(arcstat_evict_l2_eligible,
						    ab->b_size);
					} else {
						ARCSTAT_INCR(
						    arcstat_evict_l2_ineligible,
						    ab->b_size);
					}
				}
	
				if (ab->b_datacnt == 0) {
					arc_change_state(evicted_state, ab, hash_lock);
					ASSERT(HDR_IN_HASH_TABLE(ab));
					ab->b_flags |= ARC_IN_HASH_TABLE;
					ab->b_flags &= ~ARC_BUF_AVAILABLE;
					DTRACE_PROBE1(arc__evict, arc_buf_hdr_t *, ab);
				}
				if (!have_lock)
					mutex_exit(hash_lock);
				if (bytes >= 0 && bytes_evicted >= bytes)
					break;
			} else {
				missed += 1;
			}
		}
	
		mutex_exit(&evicted_state->arcs_mtx);
		mutex_exit(&state->arcs_mtx);
	
		if (bytes_evicted < bytes)
			dprintf("only evicted %lld bytes from %x",
			    (longlong_t)bytes_evicted, state);
	
		if (skipped)
			ARCSTAT_INCR(arcstat_evict_skip, skipped);
	
		if (missed)
			ARCSTAT_INCR(arcstat_mutex_miss, missed);
	
		/*
		 * We have just evicted some date into the ghost state, make
		 * sure we also adjust the ghost state size if necessary.
		 */
		if (arc_no_grow &&
		    arc_mru_ghost->arcs_size + arc_mfu_ghost->arcs_size > arc_c) {
			int64_t mru_over = arc_anon->arcs_size + arc_mru->arcs_size +
			    arc_mru_ghost->arcs_size - arc_c;
	
			if (mru_over > 0 && arc_mru_ghost->arcs_lsize[type] > 0) {
				int64_t todelete =
				    MIN(arc_mru_ghost->arcs_lsize[type], mru_over);
				arc_evict_ghost(arc_mru_ghost, 0, todelete);
			} else if (arc_mfu_ghost->arcs_lsize[type] > 0) {
				int64_t todelete = MIN(arc_mfu_ghost->arcs_lsize[type],
				    arc_mru_ghost->arcs_size +
				    arc_mfu_ghost->arcs_size - arc_c);
				arc_evict_ghost(arc_mfu_ghost, 0, todelete);
			}
		}
	
		return (stolen);
	}

```

The following is the detailed flowchart.

<img src="{{ site.baseurl }}/images/2015-12-14-1/ControlFlowGraph-arc_evict.png" alt="ZFS-FUSE Control Flow Graph arc_evict">


## ARC Read

```c

	/*
	 * "Read" the block block at the specified DVA (in bp) via the
	 * cache.  If the block is found in the cache, invoke the provided
	 * callback immediately and return.  Note that the `zio' parameter
	 * in the callback will be NULL in this case, since no IO was
	 * required.  If the block is not in the cache pass the read request
	 * on to the spa with a substitute callback function, so that the
	 * requested block will be added to the cache.
	 *
	 * If a read request arrives for a block that has a read in-progress,
	 * either wait for the in-progress read to complete (and return the
	 * results); or, if this is a read with a "done" func, add a record
	 * to the read to invoke the "done" func when the read completes,
	 * and return; or just return.
	 *
	 * arc_read_done() will invoke all the requested "done" functions
	 * for readers of this block.
	 *
	 * Normal callers should use arc_read and pass the arc buffer and offset
	 * for the bp.  But if you know you don't need locking, you can use
	 * arc_read_bp.
	 */
	int arc_read(zio_t *pio, spa_t *spa, const blkptr_t *bp, arc_buf_t *pbuf,
	    arc_done_func_t *done, void *private, int priority, int zio_flags,
	    uint32_t *arc_flags, const zbookmark_t *zb)
	{
		int err;
	
		ASSERT(!refcount_is_zero(&pbuf->b_hdr->b_refcnt));
		ASSERT3U((char *)bp - (char *)pbuf->b_data, <, pbuf->b_hdr->b_size);
		rw_enter(&pbuf->b_lock, RW_READER);
	
		err = arc_read_nolock(pio, spa, bp, done, private, priority,
		    zio_flags, arc_flags, zb);
		rw_exit(&pbuf->b_lock);
	
		return (err);
	}

```

The following is the detailed flowchart.

<img src="{{ site.baseurl }}/images/2015-12-14-1/ControlFlowGraph-arc_read.png" alt="ZFS-FUSE Control Flow Graph arc_read">

<img src="{{ site.baseurl }}/images/2015-12-14-1/ControlFlowGraph-arc_read_nolock.png" alt="ZFS-FUSE Control Flow Graph arc_read_nolock">

## References

This blog entry contains edits based on the following sources, credits should go to these authors!

* http://dtrace.org/blogs/brendan/2012/01/09/activity-of-the-zfs-arc
* https://www.illumos.org/issues/5497
* http://m.blog.chinaunix.net/uid-24395800-id-4225348.html
* http://www.cuddletech.com/blog/pivot/entry.php?id=979
* https://www.joyent.com/blog/what-s-in-the-arc