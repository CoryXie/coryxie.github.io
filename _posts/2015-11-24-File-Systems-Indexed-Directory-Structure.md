---
layout: post
title: "File Systems Indexed Directory Structure"
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

Linear directory structure is not good for system performance when thousands or millions of files are in a single directory, where we have to use some better indexing techniques to improve directory operations. An HTree is a specialized tree data structure for directory indexing, similar to a B-tree. They are constant depth of either one or two levels, have a high fanout factor, use a hash of the filename, and do not require balancing. The HTree algorithm is distinguished from standard B-tree methods by its treatment of hash collisions, which may overflow across multiple leaf and index blocks. HTree indexes are used in the Linux ext3 and ext4 filesystems. HTree indexing improved the scalability of Linux ext2 based filesystems from a practical limit of a few thousand files, into the range of tens of millions of files per directory. This blog entry tries to understand the HTree structure of modern filesystems.

# Overview of Indexed Directory Structure

There are two types of blocks in an indexed directory:

* Leaf block (or entries block) stores directory entries (filenames)
* Tree block (or indices block) stores hash-value/block-ID pairs
  * Hash value: hash value of the entry name.
  * Block-ID: either file logical block number of leaf block, or the next level indices block number

<img src="{{ site.baseurl }}/images/2015-11-24-2/Filesystems-HashTree.png" alt="Filesystems HashTree">

This figure illustrates how storage and lookup takes place on entries in a HTree directory. Gray blocks are tree blocks, and blue blocks are leaf blocks. Value pairs in tree blocks are [hash, block-ID], while strings in leaf blocks are "filenames". For example, if hash-value of name "E52" is 155, first the top level tree block-0 is scanned and the hash value 155 (between hash 100 - 199) is found in the next level tree block-2. Searching block-2 shows that hash 155 (higher than hash 150) is in leaf block-5, where the filename "E52" can be found.

An indexed directory stores all name entries in an leaf block. A leaf block can contain between 15 - 340 entries, depending on the filename length (between 4 and 256 characters), but typically around 100 entries. It is not possible to parallelize locking at a smaller granularity than a single leaf block.

For directories with only a single leaf block, there is no tree block. When the directory grows and name entries overflow one leaf block, the filesystem will mark the directory as “indexed directory”. A this point, name entries will be sorted by hash-value and the leaf block will be split into two leaf blocks at the median hash value. A tree block will be allocated to store the [hash, block-ID] for these two leaf blocks. At this point, the structure is described as a htree.

Leaf blocks are split again and again as the size of the directory grows. Tree blocks can be split as well if the number of leaf blocks increases to the point where their indices overflow one tree block (more than 510 indices).

# EXT4 Implementation of HTree

A linear array of directory entries isn't great for performance, so a new feature was added to ext3 to provide a faster (but peculiar) balanced tree keyed off a hash of the directory entry name. If the `EXT4_INDEX_FL (0x1000)` flag is set in the inode, this directory uses a hashed btree (htree) to organize and find directory entries. For backwards read-only compatibility with ext2, this tree is actually hidden inside the directory file, masquerading as "empty" directory data blocks! It was stated previously that the end of the linear directory entry table was signified with an entry pointing to inode 0; this is (ab)used to fool the old linear-scan algorithm into thinking that the rest of the directory block is empty so that it moves on.

* The root of the tree always lives in the first data block of the directory. By ext2 custom, the '.' and '..' entries must appear at the beginning of this first block, so they are put here as two `struct ext4_dir_entry_2`s and not stored in the tree. The rest of the root node contains metadata about the tree and finally a `hash -> block` map to find nodes that are lower in the htree. 
* If `dx_root.info.indirect_levels` is non-zero then the htree has two levels; the data block pointed to by the root node's map is an interior node, which is indexed by a minor hash. Interior nodes in this tree contains a zeroed out `struct ext4_dir_entry_2` followed by a `minor_hash -> block` map to find leaf nodes. 
* Leaf nodes contain a linear array of all `struct ext4_dir_entry_2`; all of these entries (presumably) hash to the same value. If there is an overflow, the entries simply overflow into the next leaf node, and the least-significant bit of the hash (in the interior node map) that gets us to this next leaf node is set.

To traverse the directory as a htree, the code calculates the hash of the desired file name and uses it to find the corresponding block number. If the tree is flat, the block is a linear array of directory entries that can be searched; otherwise, the minor hash of the file name is computed and used against this second block to find the corresponding third block number. That third block number will be a linear array of directory entries.

To traverse the directory as a linear array (such as the old code does), the code simply reads every data block in the directory. The blocks used for the htree will appear to have no entries (aside from '.' and '..') and so only the leaf nodes will appear to have any interesting content.

The root of the htree is in `struct dx_root`, which is the full length of a data block:

<img src="{{ site.baseurl }}/images/2015-11-24-2/ext4-dx_root.png" alt="EXT4 Filesystems dx_root">

Interior nodes of an htree are recorded as `struct dx_node`, which is also the full length of a data block:

<img src="{{ site.baseurl }}/images/2015-11-24-2/ext4-dx_node.png" alt="EXT4 Filesystems dx_node">

The hash maps that exist in both `struct dx_root` and `struct dx_node` are recorded as `struct dx_entry`, which is 8 bytes long:

<img src="{{ site.baseurl }}/images/2015-11-24-2/ext4-dx_entry.png" alt="EXT4 Filesystems dx_entry">

(If you think this is all quite clever and peculiar, so does the author.)

If metadata checksums are enabled, the last 8 bytes of the directory block (precisely the length of one `dx_entry`) are used to store a `struct dx_tail`, which contains the checksum. The limit and count entries in the `dx_root`/`dx_node` structures are adjusted as necessary to fit the `dx_tail` into the block. If there is no space for the `dx_tail`, the user is notified to run `e2fsck -D` to rebuild the directory index (which will ensure that there's space for the checksum. The `dx_tail` structure is 8 bytes long and looks like this:

<img src="{{ site.baseurl }}/images/2015-11-24-2/ext4-dx_tail.png" alt="EXT4 Filesystems dx_tail">

The checksum is calculated against the FS UUID, the htree index header (`dx_root` or `dx_node`), all of the htree indices (`dx_entry`) that are in use, and the tail block (`dx_tail`).

# References

This blog entry is mostly an edit of the following sources, credits should go to these authors!

* "A Directory Index for Ext2" from [https://www.kernel.org/doc/ols/2002/ols2002-pages-425-438.pdf](https://www.kernel.org/doc/ols/2002/ols2002-pages-425-438.pdf)
* "MDS PDirOps SolutionArchitecture" from [http://wiki.opensfs.org/MDS_PDirOps_SolutionArchitecture_wiki_version](http://wiki.opensfs.org/MDS_PDirOps_SolutionArchitecture_wiki_version)
*  "The new ext4 filesystem: current status and future plans" from [https://www.kernel.org/doc/ols/2007/ols2007v2-pages-21-34.pdf](https://www.kernel.org/doc/ols/2007/ols2007v2-pages-21-34.pdf)
*  "Ext4 Disk Layout" from [https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout)
*  "Ext3 Directory Index Mechanism" from [http://www.oenhan.com/ext3-dir-hash-index](http://www.oenhan.com/ext3-dir-hash-index)
