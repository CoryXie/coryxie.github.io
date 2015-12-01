---
layout: post
title: "ZFS-FUSE Project Structure"
description: 
headline: 
modified: 2015-12-01
category: FileSysDev
tags: [Filesystem]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

The ZFS-FUSE project is a port of the ZFS filesystem to FUSE/Linux, done as part of the Google Summer of Code 2006 initiative. This makes it possible to create, mount, use, and manage ZFS filesystems under Linux. ZFS itself is one of the most advanced, feature-rich file systems in existence. It incorporates variable block sizes, compression, encryption, de-duplication, snapshots, clones, and (as the name implies) supports for massive capacities. This blog post series will try to understand the concepts behind ZFS and learn how we can use ZFS on Linux using Filesystem in Userspace (FUSE).

# Build ZFS-FUSE

Original ZFS-FUSE project source code can be found [here](https://github.com/zfs-fuse/zfs-fuse) and simply do the following can get you a copy of the source tree.

 * git clone https://github.com/zfs-fuse/zfs-fuse.git

However, this project is not actively maintained (the most recent commit was made on Feb 1, 2012, it was years back), thus we are expected to see some diffs when building and running it on latest Linux distributions.

According to the INSTALL document in zfs-fuse, the following packages are required:

 * scons
 * libfuse-dev (>= 2.8.1)
 * zlib1g-dev
 * libaio-dev
 * libssl-dev
 * libattr1-dev

On a Ubutun 14.04 system I've managed to build it with the following commands:

 * apt-get install scons
 * apt-get install libfuse-dev
 * cd src
 * scons

One thing to note is that the exact clone from [here](https://github.com/zfs-fuse/zfs-fuse) would have the following issues when build:

```console

	gcc -DHAVE_CONFIG_H -I. -I. -I. -g -O2 -MT malloc.lo -MD -MP -MF .deps/malloc.Tpo -c malloc.c  -fPIC -DPIC -o .libs/malloc.o
	malloc.c: In function 'umem_malloc_init_hook':
	malloc.c:447:2: warning: '__malloc_hook' is deprecated (declared at /usr/include/malloc.h:176) [-Wdeprecated-declarations]
	  if (__malloc_hook != umem_malloc_hook) {
	  ^
	malloc.c:449:3: warning: '__malloc_hook' is deprecated (declared at /usr/include/malloc.h:176) [-Wdeprecated-declarations]
	   __malloc_hook = umem_malloc_hook;
	   ^
	malloc.c:450:3: warning: '__free_hook' is deprecated (declared at /usr/include/malloc.h:173) [-Wdeprecated-declarations]
	   __free_hook = umem_free_hook;
	   ^
	malloc.c:451:3: warning: '__realloc_hook' is deprecated (declared at /usr/include/malloc.h:179) [-Wdeprecated-declarations]
	   __realloc_hook = umem_realloc_hook;
	   ^
	malloc.c:452:3: warning: '__memalign_hook' is deprecated (declared at /usr/include/malloc.h:183) [-Wdeprecated-declarations]
	   __memalign_hook = umem_memalign_hook;
	   ^
	malloc.c: At top level:
	malloc.c:456:8: error: conflicting type qualifiers for '__malloc_initialize_hook'
	 void (*__malloc_initialize_hook)(void) = umem_malloc_init_hook;
	        ^
	In file included from malloc.c:45:0:
	/usr/include/malloc.h:170:38: note: previous declaration of '__malloc_initialize_hook' was here
	 extern void (*__MALLOC_HOOK_VOLATILE __malloc_initialize_hook) (void)
	                                      ^
	make[1]: *** [malloc.lo] Error 1
	make[1]: Leaving directory `/buildarea1/wxie/projects/zfs-fuse/src/lib/libumem'

```

I've made the fix by the following change in `zfs-fuse/src/lib/libumem/malloc.c` and pushed into my own [fork](https://github.com/CoryXie/zfs-fuse).

```console
	
	-void (*__malloc_initialize_hook)(void) = umem_malloc_init_hook;
	+void (*__MALLOC_HOOK_VOLATILE __malloc_initialize_hook)(void) = umem_malloc_init_hook;

```

# ZFS-FUSE Project Strucure

I've run Doxygen under zfs-fuse directory and genrated the document of this project. I will try to use this document to understand the overal project strcture. 

Since zfs-fuse is based on FUSE, and it exhibts several user space programs to manage the filesystem, there must be some main() entry points to be used as starting points for us to navigate.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-main-entry-points.png" alt="ZFS-FUSE Main Entry Points">

The `zfs-fuse/src/cmd` directory contains several user space programs for use with ZFS-FUSE. 

* Command `zdb` as ZFS debugger. 

The `zdb` command is used by support engineers to diagnose failures and gather statistics. Since the ZFS file system is always consistent on disk and is self-repairing, zdb should only be run under the direction by a support engineer. If no arguments are specified, `zdb`, performs basic consistency checks on the pool and associated datasets, and report any problems detected.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-cmd-zdb-sources.png" alt="ZFS-FUSE zdb Command Sources">

* Command `zfs` configures ZFS file systems.

The `zfs` command configures ZFS datasets within a ZFS storage pool, as described in `zpool`. A dataset is identified by a unique path within the ZFS namespace. For example, `pool/{filesystem,volume,snapshot}`, where the maximum length of a dataset name is `MAXNAMELEN` (256 bytes). A dataset can be one of the following:

>- file system

A ZFS dataset of type filesystem can be mounted within the standard system namespace and behaves like other file systems. While ZFS file systems are designed to be POSIX compliant, known issues exist that prevent compliance in some cases. Applications that depend on standards conformance might fail due to nonstandard behavior when checking file system free space.

>- volume

A logical volume exported as a raw or block device. This type of dataset should only be used under special circumstances. File systems are typically used in most environments.

>- snapshot

A read-only version of a file system or volume at a given point in time. It is specified as `filesystem@name` or `volume@name`.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-cmd-zfs-sources.png" alt="ZFS-FUSE zfs Command Sources">

* Command `zpool` configures ZFS storage pools.

The `zpool` command configures ZFS storage pools. A storage pool is a collection of devices that provides physical storage and data replication for ZFS datasets. All datasets within a storage pool share the same space.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-cmd-zpool-sources.png" alt="ZFS-FUSE zpool Command Sources">

* Command `zstreamdump` filters data in `zfs send` stream.

The `zstreamdump` utility reads from the output of the `zfs send` command, then displays headers and some statistics from that output.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-cmd-zstreamdump-sources.png" alt="ZFS-FUSE zstreamdump Command Sources">

* Command `ztest` was written by the ZFS Developers as a ZFS unit test.

`ztest` was written by the ZFS Developers as a ZFS unit test. The tool was developed in tandem with the ZFS functionality and was executed nightly as one of the many regression test against the daily build. As features were added to ZFS, unit tests were also added to ztest. In addition, a separate test development team wrote and executed more functional and stress tests.

By default `ztest` runs for ten minutes and uses block files (stored in `/tmp`) to create pools rather than using physical disks. Block files afford `ztest` its flexibility to play around with `zpool` components without requiring large hardware configurations. However, storing the block files in `/tmp` may not work for you if you have a small `/tmp` directory.

By default it is non-verbose. This is why entering the command above will result in `ztest` quietly executing for 5 minutes. The `-V` option can be used to increase the verbosity of the tool. Adding multiple `-V` option is allowed and the more you add the more chatty `ztest` becomes.

After the `ztest` run completes, you should notice many `ztest.*` files lying around. Once the run completes you can safely remove these files. Note that you shouldn't remove these files during a run. You can re-use these files in your next *`ztest`* run by using the `-E` option.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-cmd-ztest-sources.png" alt="ZFS-FUSE ztest Command Sources">

* Library `libavl` is a generic AVL tree implementation for kernel use (but ported to user space in `zfs-fuse`).

An AVL tree is a binary search tree that is almost perfectly balanced. By "almost" perfectly balanced, we mean that at any given node, the left and right subtrees are allowed to differ in height by at most 1 level.
 
This relaxation from a perfectly balanced binary tree allows doing insertion and deletion relatively efficiently. Searching the tree is still a fast operation, roughly O(log(N)).
 
The key to insertion and deletion is a set of tree manipulations called rotations, which bring unbalanced subtrees back into the semi-balanced state.
 
This implementation of AVL trees has the following peculiarities:
 
 > - The AVL specific data structures are physically embedded as fields in the "using" data structures.  To maintain generality the code must constantly translate between `avl_node_t *` and containing data structure `void *` by adding/subracting the `avl_offset`.
 
 > - Since the AVL data is always embedded in other structures, there is no locking or memory allocation in the AVL routines. This must be provided for by the enclosing data structure's semantics. Typically, `avl_insert()/_add()/_remove()/avl_insert_here()` require some kind of exclusive write lock. Other operations require a read lock.
 
 > - The implementation uses iteration instead of explicit recursion, since it is intended to run on limited size kernel stacks. Since there is no recursion stack present to move "up" in the tree, there is an explicit "parent" link in the `avl_node_t`.
 
 > - The left/right children pointers of a node are in an array. In the code, variables (instead of constants) are used to represent left and right indices. The implementation is written as if it only dealt with left handed manipulations.  By changing the value assigned to "left", the code also works for right handed trees. Though it is a little more confusing to read the code, the approach allows using half as much code (and hence cache footprint) for tree manipulations and eliminates many conditional branches. The following variables/terms are frequently used:
 
```c

 		int left;	// 0 when dealing with left children,
 				    // 1 for dealing with right children
 
 		int left_heavy;	// -1 when left subtree is taller at some node,
 				        // +1 when right subtree is taller
 
 		int right;	    // will be the opposite of left (0 or 1)
 		int right_heavy;// will be the opposite of left_heavy (-1 or 1)
 
 		int direction;  // 0 for "<" (ie. left child); 1 for ">" (right)
```
 
 > - The `avl_index_t` is an opaque "cookie" used to find nodes at or adjacent to where a new value would be inserted in the tree. The value is a modified `avl_node_t *`. The bottom bit (normally 0 for a pointer) is set to indicate if that new node has a value greater than the value of the indicated `avl_node_t *`.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libavl-sources.png" alt="ZFS-FUSE libavl Library Sources">

* Library `libnvpair` is a name-value pair library.

The `libnvpair` library exports a set of functions for managing name-value pairs. The library defines four opaque handles:

 > - `nvpair_t` 	 handle to a name-value pair
 > - `nvlist_t` 	 handle to a list of name-value pairs
 > - `nv_alloc_t`	 handle to a pluggable allocator
 > - `nv_alloc_ops_t`	 handle to pluggable allocator operations

The library supports the following operations:

 > - Allocate and free an `nvlist_t`.
 > - Specify the allocater to be used when manipulating an `nvlist_t`.
 > - Add and remove an `nvpair_t` from a list.
 > - Search `nvlist_t` for a specified name pair.
 > - Pack an `nvlist_t` into a contiguous buffer.
 > - Expand a packed nvlist into a searchable `nvlist_t`.


<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libnvpair-sources.png" alt="ZFS-FUSE libnvpair Library Sources">

* Library `libsolcompat` implements or translates all necessary Solaris user space specific functions into glibc functions.

This `libsolcompat` makes all `#include`s to be exactly the same between the original ZFS source files and the Linux port.

This was achieved by overriding some glibc system headers with a local header, with the help of a gcc-specific preprocessor directive called `#include_next`. This directive allows one to include the overriden system header, while adding some functionality to it.

For example, the `libsolcompat` has a `src/lib/libsolcompat/include/string.h`, where a Solaris-specific function called `strlcpy()` was added, and implemented in `src/lib/libsolcompat/strlcat.c`.

```c

	#ifndef _SOL_STRING_H
	#define _SOL_STRING_H
	
	#include_next <string.h>
	
	extern size_t strlcpy(char *dst, const char *src, size_t len);
	extern size_t strlcat(char *, const char *, size_t);
	
	#endif

```

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolcompat-sources-1.png" alt="ZFS-FUSE libsolcompat Library Sources 1">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolcompat-sources-2.png" alt="ZFS-FUSE libsolcompat Library Sources 2">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolcompat-sources-3.png" alt="ZFS-FUSE libsolcompat Library Sources 3">

* Library `libsolkerncompat` implements or translates all necessary Solaris kernel space specific functions into glibc functions.

The `libsolkerncompat` library implements or translates the necessary OpenSolaris kernel code to make the `ZPL` work. This library will also be necessary in order to use the original `zfs_ioctl.c` implementation, since it uses some kernel `VFS` (Virtual File System) operations, along with other things. A new `zfs_context.h` was also created or (more likely) the existing OpenSolaris `zfs_context.h` was factored out to `libsolkerncompat`.

Note that the `ZPL` (ZFS POSIX Layer) runs at a high level, taking instruction from the OS on I/O requests. Below that is the `DMU` (Data Management Unit) that takes those instructions and translates them into transaction batches. Rather than requesting data blocks and sending single write requests, ZFS batches these into object-based transactions that can be optimized before any disk activity occurs.

Once this is done, the batches are handed off to the `SPA` (Storage Pool Allocator) to schedule and aggregate the raw I/O. The `copy-on-write` basis of I/O transactions, coupled with checksums performed on a per block basis, precludes the need for journaling. An abrupt power loss will be recoverable at any point. 

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolkerncompat-sources-1.png" alt="ZFS-FUSE libsolkerncompat Library Sources 1">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolkerncompat-sources-2.png" alt="ZFS-FUSE libsolkerncompat Library Sources 2">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolkerncompat-sources-3.png" alt="ZFS-FUSE libsolkerncompat Library Sources 3">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolkerncompat-sources-4.png" alt="ZFS-FUSE libsolkerncompat Library Sources 4">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libsolkerncompat-sources-5.png" alt="ZFS-FUSE libsolkerncompat Library Sources 5">

* Library `libumem` is a library used to detect memory management bugs in applications.

`Libumem` is based on the Slab allocator concept. Libumem is available as a standard part of Solaris from Solaris 9 Update 3 onwards. 

Functions in this library provide fast, scalable object-caching memory allocation with multithreaded application support. In addition to the standard `malloc()` family of functions and the more flexible `umem_alloc()` family, `libumem` provides powerful object-caching services as described in `umem_cache_create()`.

Getting started with `libumem` is easy; just set `LD_PRELOAD` to "libumem.so" and any program executed will use `libumem`'s `malloc()` and `free()` (or `new` and `delete`). This slab allocator is designed for systems with many threads and many CPUs. Memory allocation with naive allocators can be a serious bottleneck.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libumem-sources.png" alt="ZFS-FUSE libumem Library Sources">

* Library `libuutil` is a library of userland utilities originating from OpenSolaris.

This library provides both a doubly linked-list implementation and a AVL tree implementation. This has been a private library best known as a core component for `ZFS` and `SMF`.

The performance is considered excellent. As this has always been a private library, it is not well documented and there is no man page for it. The best documentation is located in the source code and reading OpenSolaris/Illumos `ZFS` and `SMF` sources will help as well.

For example, the list implementation is mostly located in `src/lib/libuutil/common/uu_list.c`.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libuutil-sources.png" alt="ZFS-FUSE libuutil Library Sources">

* Library `libzfs` is used by the `zfs` and `zpool` administration tools to communicate with ZFS code in the kernel.
 
This communication occurs . This library provides a unified, object-based mechanism  for accessing and processing storage pool and file system. The underlying mechanism which communicate with the kernel implements as `ioctl()` syscalls on `/dev/zfs`. It contains the following files:

> - libzfs_dataset.c	Key interfaces for processing data sets
> - libzfs_pool.c	Primary interfaces for processing storage pool
> - libzfs_changelist.c	Attribute change utility
> - libzfs_config.c	Read and process the storage pool configuration information
> - libzfs_graph.c	Build a list of related data collection
> - libzfs_import.c	Search and import storage pool
> - libzfs_mount.c	Mount, unmount and share datasets
> - libzfs_status.c	The storage pool state linked to the FMA Knowledge Base
> - libzfs_util.c	Other routines

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libzfs-sources.png" alt="ZFS-FUSE libzfs Library Sources">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libzfscommon-sources-1.png" alt="ZFS-FUSE libzfscommon Library Sources 1">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libzfscommon-sources-2.png" alt="ZFS-FUSE libzfscommon Library Sources 2">

* Library `libzpool` contains the userland port of the `SPA` (Storage Pool Allocator) and `DMU` (Data Management Unit) code.

ZFS file system ships with a userspace testing library, `libzpool`. In addition to kernel interface emulation routines, `libzpool` consists of the Data Management Unit and Storage Pool Allocator components of ZFS compiled from the kernel sources. The `ztest` program plugs directly to these components.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libzpool-sources-1.png" alt="ZFS-FUSE libzpool Library Sources 1">

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-lib-libzpool-sources-2.png" alt="ZFS-FUSE libzpool Library Sources 2">

* Core `zfs-fuse` FUSE code

FUSE is a mechanism that allows you to implement file systems in user space without kernel code (other than the FUSE kernel module and existing file system code). The module provides a bridge from the kernel file system interface to user space for user and file system implementations. 

`zfs-fuse` is the main `zfs-fuse` daemon and must be running at all times. It should be run with root privileges.

<img src="{{ site.baseurl }}/images/2015-12-01-1/zfs-fuse-core-fuse-sources.png" alt="ZFS-FUSE Core FUSE Sources">

We can install zfs-fuse on Ubuntu with the following command:

> - sudo apt-get install zfs-fuse

This command line install ZFS-FUSE and all other dependent packages as well as performing the necessary setup for the new packages and starting the zfs-fuse daemon.

# References

* "Run ZFS on Linux" from [IBM DeveloperWorks](http://www.ibm.com/developerworks/library/l-zfs/).
* "ZFS-FUSE source code" from [GitHub](https://github.com/zfs-fuse/zfs-fuse).
* "Oracle ZFS commands" from [Oracle](http://docs.oracle.com/cd/E23824_01/html/821-1462/toc.html).

