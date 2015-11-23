---
layout: post
title: "File Systems Indirect Blocks and Extents"
description:
headline:
modified: 2015-11-24
category: FileSysDev
tags: [Filesystem]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

Questions always come up about the function of **indirect blocks** in standard Unix file systems (meaning ext2/ext3 on Linux, UFS on Solaris, FFS on BSD, and others derived, directly or indirectly). Modern file systems deal with larger than 4TB files by recognizing that addressing every single block in the file is incredibly wasteful. Instead the file system allocates collections of consecutive blocks, usually referred to as **extents**, and then stores pointers to the first block in the extents. This blog entry tries to understand these basic filesystem concepts.

# Indirect Blocks

In a standard Unix file system, files are made up of two different types of objects. Every file has an **index node** (inode for short) associated with it that contains the **metadata** about that file: permissions, ownerships, timestamps, etc. The contents of the file are stored in a collection of **data blocks**. At this point in the discussion, a lot of people just wave their hands and say something like, "And there are pointers in the inode that link to the data blocks."

As it turns out, there are only **fifteen** block pointers in the inode. Assuming standard 4K data blocks, that means that the largest possible file that could be addressed directly would be 60K-- obviously not nearly large enough. In fact, only the first 12 block pointers in the inode are reserved for direct block pointers. This means you can address files of up to 48K just using the direct pointers in the inode.

Beyond that, you start getting into indirect blocks:

* The 13th pointer is the indirect block pointer. Once the file grows beyond 48K, the file system grabs a data block and starts using it to store additional block pointers, setting the thirteenth block pointer in the inode to the address of this block. Block pointers are 4-byte quantities, so the indirect block can store 1024 of them. That means that the total file size that can be addressed via the indirect block is 4MB (plus the 48K of storage addressed by the direct blocks in the inode).

* Once the file size grows beyond 4MB + 48KB, the file system starts using **doubly indirect blocks**. The 14th block pointer points to a data block that contains the addresses of other indirect blocks, which in turn contain the addresses of the actual data blocks that make up the file's contents. That means we have up to 1024 indirect blocks that in turn point to up to 1024 data blocks -- in other words up to 1M total 4K blocks, or up to 4GB of storage.

* At this point, you've probably figured out that the 15th inode pointer is the **trebly indirect block** pointer. With three levels of indirect blocks, you can address up to 4TB (+4GB from the **doubly indirect** pointer, +4M from the **indirect block** pointer, +48K from the **direct block** pointers) for a single file.

Here's a picture to help you visualize what I'm talking about here:

<img src="{{ site.baseurl }}/images/2015-11-24-1/IndirectBlocks1.png" alt="Filesystems Indirect Blocks">

Here's another picture to help you visualize what I'm talking about here:

<img src="{{ site.baseurl }}/images/2015-11-24-1/IndirectBlocks2.png" alt="Another Filesystems Indirect Blocks">

By the way, while 4TB+ seemed like an impossibly large file back in the 1970s, these days people actually want to create files that are significantly larger than 4TB. 

# Extents

Indirect block maps are incredibly inefficient for large files becasue it needs one extra block read (and seek) every 1024 blocks, which is really obvious when deleting big CD/DVD image files. Modern file systems deal with larger than 4TB files by recognizing that addressing every single block in the file is incredibly wasteful. Instead the file system allocates collections of consecutive blocks, usually referred to as **extents**, and then stores pointers to the first block in the extents. Even with a relatively small extent size, you can index huge files -- 64K extents mean you can create a 64TB file! Using Extents is a efficient way to represent large files, and it can achieve better CPU utilization, fewer metadata IOs.

For example, for ext4 filesystem, on-­disk extent format is a 12 bytes `ext4_extent` structure:

* address 1EB filesystem (48 bit physical block number)
* max extent 128MB (15 bit extent length)
* address 16TB file size (32 bit logical block number)

```c

        struct ext4_extent {
        __le32  ee_block;   /* first logical block extent covers */
        __le16  ee_len;      /* number of blocks covered by extent */
        __le16  ee_start_hi;    /* high 16 bits of physical block */
        __le32  ee_start;     /* low 32 bits of physical block */
        };

```

Here's a picture for a Extent Map:

<img src="{{ site.baseurl }}/images/2015-11-24-1/ExtentMap.png" alt="Filesystems Extent Map">

For ext4 filesystem:

* Up to 3 extents could be stored in inode `i_data` body directly
* Use a **inode flag** to mark extents file vs ext3 indirect block file
* Convert to a B­Tree extents tree, for more than 3 extents
* Last found extent is cached in ­memory as extents tree

<img src="{{ site.baseurl }}/images/2015-11-24-1/ExtentTree.png" alt="Filesystems Extent Tree">

# References

This blog entry is mostly an edit of the following sources, credits should go to these authors!

* "Understanding Indirect Blocks in Unix File Systems" from [https://digital-forensics.sans.org/blog/2008/12/24/understanding-indirect-blocks-in-unix-file-systems](https://digital-forensics.sans.org/blog/2008/12/24/understanding-indirect-blocks-in-unix-file-systems)
* "Ext4: The Next Generation of Ext2/3 Filesystem" from [http://www.cs.loyola.edu/~binkley/466/ext4-slides.pdf](http://www.cs.loyola.edu/~binkley/466/ext4-slides.pdf)
* "An analysis of Ext4 for digital forensics" from [http://www.dfrws.org/2012/proceedings/DFRWS2012-13.pdf](http://www.dfrws.org/2012/proceedings/DFRWS2012-13.pdf)