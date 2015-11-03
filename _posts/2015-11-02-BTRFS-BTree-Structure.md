---
layout: post
title: "BTRFS BTree Structure"
description:
headline:
modified: 2015-11-02
category: BtrfsDev
tags: [Filesystem]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

BTRFS is the most promising next generation filesystem that supports a lot of features, and it has draw my interests for quite a lot of time. Since I am not originally a filesystem developer, I am going to start learning the basics of filesystem creation from scratch. The first thing come into my mind is how does a filesystem organize various information in the backing storage, for both meta data and user data, after all, data is what a filesystem actually used for. To this end, I am going to read the `btrfs-progs` source code, from its very 1st commit, in order to understand how BTRFS was originally born.

# Where Do I Start?

The `btrfs-progs` is a set of user space programs to be used with BTRFS. It can be obtained from GitHub:

```console

	git clone http://github.com/kdave/btrfs-progs.git

```

The very 1st commit by the creator *Chris Mason* has only 1 file called `ctree.c`, which has comment `Initial checkin, basic working tree code`. And the 2nd commit adds one missing header file ``, and updates a bit to `Faster deletes, add Makefile and kerncompat.h`. So with the two initial commits, it is able to compile and run with basic tree management. 

```console

	$ ls
	Makefile*  ctree.c*  kerncompat.h*

```

Since it is a user space program, the entry point is from `main()`:

```c

	int main() {
		struct leaf *first_node = malloc(sizeof(struct leaf));
		struct ctree_root root;
		struct key ins;
		char *buf;
		int i;
		int num;
		int ret;
		int run_size = 10000000;
		int max_key = 100000000;
		int tree_size = 0;
		struct ctree_path path;
	
	
		srand(55);
		root.node = (struct node *)first_node;
		memset(first_node, 0, sizeof(*first_node));
		for (i = 0; i < run_size; i++) {
			buf = malloc(64);
			num = next_key(i, max_key);
			// num = i;
			sprintf(buf, "string-%d", num);
			// printf("insert %d\n", num);
			ins.objectid = num;
			ins.offset = 0;
			ins.flags = 0;
			ret = insert_item(&root, &ins, buf, strlen(buf));
			if (!ret)
				tree_size++;
		}
		srand(55);
		for (i = 0; i < run_size; i++) {
			num = next_key(i, max_key);
			ins.objectid = num;
			ins.offset = 0;
			ins.flags = 0;
			init_path(&path);
			ret = search_slot(&root, &ins, &path);
			if (ret) {
				print_tree(root.node);
				printf("unable to find %d\n", num);
				exit(1);
			}
		}
		printf("node %p level %d total ptrs %d free spc %lu\n", root.node,
		        node_level(root.node->header.flags), root.node->header.nritems,
			NODEPTRS_PER_BLOCK - root.node->header.nritems);
		// print_tree(root.node);
		printf("all searches good\n");
		i = 0;
		srand(55);
		for (i = 0; i < run_size; i++) {
			num = next_key(i, max_key);
			ins.objectid = num;
			del_item(&root, &ins);
		}
		print_tree(root.node);
		return 0;
	}

```

As we can see, this program only tries to verify the BTree algorithm actually works by performing the following steps.


1) Create the root node.

```c

	struct leaf *first_node = malloc(sizeof(struct leaf));
	root.node = (struct node *)first_node;

```

2) Insert items into the BTree represented by the root node.

```c

		for (i = 0; i < run_size; i++) {
			buf = malloc(64);
			num = next_key(i, max_key);
			// num = i;
			sprintf(buf, "string-%d", num);
			// printf("insert %d\n", num);
			ins.objectid = num;
			ins.offset = 0;
			ins.flags = 0;
			ret = insert_item(&root, &ins, buf, strlen(buf));
			if (!ret)
				tree_size++;
		}

```

3) Search items from the BTree represented by the root node, and if any search fails, exit the program.

```c

		for (i = 0; i < run_size; i++) {
			num = next_key(i, max_key);
			ins.objectid = num;
			ins.offset = 0;
			ins.flags = 0;
			init_path(&path);
			ret = search_slot(&root, &ins, &path);
			if (ret) {
				print_tree(root.node);
				printf("unable to find %d\n", num);
				exit(1);
			}
		}

```

4) Delete items from the BTree represented by the root node, and prints the tree so that it is visually clear the tree is empty again (since all nodes have been deleted).

```c

		for (i = 0; i < run_size; i++) {
			num = next_key(i, max_key);
			ins.objectid = num;
			del_item(&root, &ins);
		}
		print_tree(root.node);

```

So we will analyze the BTree inserting, searching, deleting code in the following sections.

# BTree Data Structures

A **multiway tree of order m** is an ordered tree where each node has at most m children. For each node, if k is the actual number of children in the node, then k - 1 is the number of keys in the node. If the keys and subtrees are arranged in the fashion of a search tree, then this is called a **multiway search tree of order m**.

A B-tree of order m is a multiway search tree of order m such that:

* All leaves are on the bottom level.
* All internal nodes (except perhaps the root node) have at least `ceil(m/2)` (nonempty) children.
* The root node can have as few as 2 children if it is an internal node, and can obviously have no children if the root node is a leaf (that is, the whole tree consists only of the root node).
* Each leaf node (other than the root node if it is a leaf) must contain at least `ceil(m/2)-1` keys.

The link [http://cis.stvincent.edu/html/tutorials/swd/btree/btree.html](http://cis.stvincent.edu/html/tutorials/swd/btree/btree.html) contains details about B-tree.

A BTree in BTRFS user space program is represented by an instance of `struct ctree_root`, which has a single pointer to `struct node`.

```c

	struct ctree_root {
		struct node *node;
	};

```

Each `struct node` occupies a `BLOCKSIZE` of disk space, which is defined to be 4KB, thus a `struct node` is effectively representing a `BLOCK`.

```c

	struct node {
		struct header header;
		struct key keys[NODEPTRS_PER_BLOCK];
		u64 blockptrs[NODEPTRS_PER_BLOCK];
	} __attribute__ ((__packed__));

	#define BLOCKSIZE 4096
	
	struct key {
		u64 objectid;
		u32 flags;
		u64 offset;
	} __attribute__ ((__packed__));
	
	struct header {
		u64 fsid[2]; /* FS specific uuid */
		u64 blocknum;
		u64 parentid;
		u32 csum;
		u32 ham;
		u16 nritems;
		u16 flags;
	} __attribute__ ((__packed__));
	
	#define NODEPTRS_PER_BLOCK ((BLOCKSIZE - sizeof(struct header)) / \
				    (sizeof(struct key) + sizeof(u64)))
	
```

The following is a graphical representation of `struct node`.

<img src="{{ site.baseurl }}/images/2015-11-02-1/btrfs-struct-node.png" alt="btrfs struct node">

Note that there maybe a small gap between the `keys[]` array and `blockptrs[]` array, and a small gap in the end of the `BLOCK`. Adding the following print code will show the exactly sizes.

```c

   printf("sizeof(struct node)= %d, sizeof(struct header)=%d, sizeof(struct key)=%d,\nNODEPTRS_PER_BLOCK=%d, sizeof(struct leaf)=%d, sizeof(struct item)=%d\n", sizeof(struct node), sizeof(struct header), sizeof(struct key), NODEPTRS_PER_BLOCK, sizeof(struct leaf), sizeof(struct item));

```

```console

	sizeof(struct node)=4076, sizeof(struct header)=44, sizeof(struct key)=20,
	NODEPTRS_PER_BLOCK=144, sizeof(struct leaf)=4096, sizeof(struct item)=24

```

The `struct node` is used for internal nodes of the BTree, while the `struct leaf` is used for leaf nodes of the BTree (as the name suggests). Since `struct leaf` uses an union for its data area, it will occupy a total of BLOCKSIZE (4KB). The `NODEPTRS_PER_BLOCK=144` means the internal nodes have at most 144 children (order 144).

```c

	struct item {
		struct key key;
		u16 offset;
		u16 size;
	} __attribute__ ((__packed__));
	
	#define LEAF_DATA_SIZE (BLOCKSIZE - sizeof(struct header))
	struct leaf {
		struct header header;
		union {
			struct item items[LEAF_DATA_SIZE/sizeof(struct item)];
			u8 data[BLOCKSIZE-sizeof(struct header)];
		};
	} __attribute__ ((__packed__));

```

The following is a graphical representation of `struct leaf`.

<img src="{{ site.baseurl }}/images/2015-11-02-1/btrfs-struct-leaf.png" alt="btrfs struct leaf">

With these nodes defined, we can have a graphical representation of a B-Tree.

<img src="{{ site.baseurl }}/images/2015-11-02-1/btrfs-btree-graph.png" alt="btrfs btree graph">

Note that I am using "..." to represent some similar node connections to save space.

# Printing Items of BTree

Without knowing how the BTree is constructed, we can see how the BTree is traversed when printing the BTree represented by the root node.

```c

	void print_tree(struct node *c)
	{
		int i;
		int nr;
	
		if (!c)
			return;
		nr = c->header.nritems;
		if (is_leaf(c->header.flags)) {
			print_leaf((struct leaf *)c);
			return;
		}
		printf("node %p level %d total ptrs %d free spc %lu\n", c,
		        node_level(c->header.flags), c->header.nritems,
			NODEPTRS_PER_BLOCK - c->header.nritems);
		fflush(stdout);
		for (i = 0; i < nr; i++) {
			printf("\tkey %d (%lu %u %lu) block %lx\n",
			       i,
			       c->keys[i].objectid, c->keys[i].flags, c->keys[i].offset,
			       c->blockptrs[i]);
			fflush(stdout);
		}
		for (i = 0; i < nr; i++) {
			struct node *next = read_block(c->blockptrs[i]);
			if (is_leaf(next->header.flags) &&
			    node_level(c->header.flags) != 1)
				BUG();
			if (node_level(next->header.flags) !=
				node_level(c->header.flags) - 1)
				BUG();
			print_tree(next);
		}
	
	}

	void print_leaf(struct leaf *l)
	{
		int i;
		int nr = l->header.nritems;
		struct item *item;
		printf("leaf %p total ptrs %d free space %d\n", l, nr,
		       leaf_free_space(l));
		fflush(stdout);
		for (i = 0 ; i < nr ; i++) {
			item = l->items + i;
			printf("\titem %d key (%lu %u %lu) itemoff %d itemsize %d\n",
				i,
				item->key.objectid, item->key.flags, item->key.offset,
				item->offset, item->size);
			fflush(stdout);
			printf("\t\titem data %.*s\n", item->size, l->data+item->offset);
			fflush(stdout);
		}
	}

```

We can note that the `flags` filed of `struct header` is used to keep track of the node level it is on (in its lower `LEVEL_BITS`, with a MAX of 8 levels). We can also see that the `node_level(f)` is used to get the current level of a node, and `is_leaf(f)` is used to check if the current node is a leaf node (which has a level of 0).

```c

	#define LEVEL_BITS 3
	#define MAX_LEVEL (1 << LEVEL_BITS)
	#define node_level(f) ((f) & (MAX_LEVEL-1))
	#define is_leaf(f) (node_level(f) == 0)

```

The transversal is a typical depth first search method, with the leaf node being the termination of a branch transversal. 

# Searching Items from BTree

```c

	int search_slot(struct ctree_root *root, struct key *key, struct ctree_path *p)
	{
		struct node *c = root->node;
		int slot;
		int ret;
		int level;
		while (c) {
			level = node_level(c->header.flags);
			p->nodes[level] = c;
			ret = bin_search(c, key, &slot);
			if (!is_leaf(c->header.flags)) {
				if (ret && slot > 0)
					slot -= 1;
				p->slots[level] = slot;
				c = read_block(c->blockptrs[slot]);
				continue;
			} else {
				p->slots[level] = slot;
				return ret;
			}
		}
		return -1;
	}

```

The `struct ctree_path` is defined as below, and works as a output parameter for calling `search_slot()` to record the path from the root to the located node. The entry in `nodes[level]` keeps the node pointer of the path on `level`, and corresponding entry in `slots[level]` keeps the slot (or index) of `keys[]/blockptrs[]` array in internal node.

```c
	
	struct ctree_path {
		struct node *nodes[MAX_LEVEL];
		int slots[MAX_LEVEL];
	};

```

The search starts from the root node. It uses the `struct node *c` pointer to keep track of the node on current level to be searched, so it starts with `c = root->node`. It gets the current level and save it in the path `p->nodes[level] = c`. Then `bin_search()` is called to find a match on the current node for the search `key` since the keys in each node are in ascending order. 

```c

	int bin_search(struct node *c, struct key *key, int *slot)
	{
		if (is_leaf(c->header.flags)) {
			struct leaf *l = (struct leaf *)c;
			return generic_bin_search((void *)l->items, sizeof(struct item),
						  key, c->header.nritems, slot);
		} else {
			return generic_bin_search((void *)c->keys, sizeof(struct key),
						  key, c->header.nritems, slot);
		}
		return -1;
	}

	int generic_bin_search(char *p, int item_size, struct key *key,
			       int max, int *slot)
	{
		int low = 0;
		int high = max;
		int mid;
		int ret;
		struct key *tmp;
	
		while(low < high) {
			mid = (low + high) / 2;
			tmp = (struct key *)(p + mid * item_size);
			ret = comp_keys(tmp, key);
	
			if (ret < 0)
				low = mid + 1;
			else if (ret > 0)
				high = mid;
			else {
				*slot = mid;
				return 0;
			}
		}
		*slot = low;
		return 1;
	}
	
```

The `bin_search()` distinguishes leaf nodes and internal nodes. For internal nodes represented by `struct node`, the `keys[]` array is separate with `blockptrs[]` array, so it can just pass `c->keys` to `generic_bin_search()` for the range of keys to be compared, and the `item_size` to be `sizeof(struct key)`. For leaf nodes represented by `struct leaf`, the keys are embedded in the `items[]` array (each item has a key in its 1st filed), so it has to pass `l->items` to `generic_bin_search()` for the range of keys to be compared, and the `item_size` to be `sizeof(struct item)`. 

The `generic_bin_search()` treats the buffer pointed to by parameter `p` to be an array of key items, each key item has a size of `item_size`. The `max` parameter should be passed as the number of key items to be compared. Then it starts a typical binary search in the key items array: 1) Initialize `low=0` and `high=max` 2) Get `tmp` to be the `mid=(low + high)/2` entry 3) Compare `key` with `tmp` by `ret = comp_keys(tmp, key)` 4) Based on the comparison result, alter `low` or `high` or return `0` when a match is found (saving its location in `slot`) 5) When the while loop fails to find a match, it returns `1` (saving `low` in the `slot`).

# Inserting Items into BTree

The insertion of the BTree is done by `insert_item()`.

```c

	int insert_item(struct ctree_root *root, struct key *key,
				  void *data, int data_size)
	{
		int ret;
		int slot;
		struct leaf *leaf;
		unsigned int nritems;
		unsigned int data_end;
		struct ctree_path path;
	
		init_path(&path);
		ret = search_slot(root, key, &path);
		if (ret == 0)
			return -EEXIST;
	
		leaf = (struct leaf *)path.nodes[0];
		if (leaf_free_space(leaf) <  sizeof(struct item) + data_size)
			split_leaf(root, &path, data_size);
		leaf = (struct leaf *)path.nodes[0];
		nritems = leaf->header.nritems;
		data_end = leaf_data_end(leaf);
		if (leaf_free_space(leaf) <  sizeof(struct item) + data_size)
			BUG();
	
		slot = path.slots[0];
		if (slot == 0)
			fixup_low_keys(&path, key, 1);
		if (slot != nritems) {
			int i;
			unsigned int old_data = leaf->items[slot].offset +
						leaf->items[slot].size;
	
			/*
			 * item0..itemN ... dataN.offset..dataN.size .. data0.size
			 */
			/* first correct the data pointers */
			for (i = slot; i < nritems; i++)
				leaf->items[i].offset -= data_size;
	
			/* shift the items */
			memmove(leaf->items + slot + 1, leaf->items + slot,
			        (nritems - slot) * sizeof(struct item));
	
			/* shift the data */
			memmove(leaf->data + data_end - data_size, leaf->data +
			        data_end, old_data - data_end);
			data_end = old_data;
		}
		memcpy(&leaf->items[slot].key, key, sizeof(struct key));
		leaf->items[slot].offset = data_end - data_size;
		leaf->items[slot].size = data_size;
		memcpy(leaf->data + data_end - data_size, data, data_size);
		leaf->header.nritems += 1;
		if (leaf_free_space(leaf) < 0)
			BUG();
		return 0;
	}

```



# Removing Items from BTree

