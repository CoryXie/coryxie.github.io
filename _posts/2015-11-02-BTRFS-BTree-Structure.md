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

The very 1st commit by the creator *Chris Mason* has only 1 file called `ctree.c`, which has comment `Initial checkin, basic working tree code`. And the 2nd commit adds one missing header file `kerncompat.h`, and updates a bit to `Faster deletes, add Makefile and kerncompat.h`. So with the two initial commits, it is able to compile and run with basic tree management. 

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

The `generic_bin_search()` treats the buffer pointed to by parameter `p` to be an array of key items, each key item has a size of `item_size`. The `max` parameter should be passed as the number of key items to be compared. Then it starts a typical binary search in the key items array: 1) Initialize `low=0` and `high=max` 2) Get `tmp` to be the `mid=(low + high)/2` entry 3) Compare `key` with `tmp` by `ret = comp_keys(tmp, key)` 4) Based on the comparison result, alter `low` or `high` or return `0` when a match is found (saving its location in `slot`) 5) When the while loop fails to find a match, it returns `1` (saving `low` in the `slot`, meaning no match found and the `slot` to be searched for the next level is at position `low`, and since the while loop can only break when `low` and `heigh` equals, it is in the middle of the node key items array).

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

The 1st thing `insert_item()` does is to make sure the tree does not already contain a node with the same `key`, so when it finds one node with `key` by calling `ret = search_slot(root, key, &path)` it simply returns `EEXIST`.

The 2nd thing `insert_item()` does is to check if the leaf has space for another data item. Note that the `path.nodes[0]` is casted to be `struct leaf` pointer, becasue level 0 is always the leaf of the B-tree. A data item should include space for a `struct item` and the buffer to hold the actual data. When data item is inserted, the `struct leaf` will have the following layout.

<img src="{{ site.baseurl }}/images/2015-11-02-1/btrfs-struct-leaf-with-data.png" alt="btrfs struct leaf with data">

The actual check for free space is done with `leaf_free_space(leaf)`, which returns the free space of the leaf, then it is compared with `sizeof(struct item) + data_size`. 

```c

	static inline unsigned int leaf_data_end(struct leaf *leaf)
	{
		unsigned int nr = leaf->header.nritems;
		if (nr == 0)
			return ARRAY_SIZE(leaf->data);
		return leaf->items[nr-1].offset;
	}
	
	static inline int leaf_free_space(struct leaf *leaf)
	{
		int data_end = leaf_data_end(leaf);
		int nritems = leaf->header.nritems;
		char *items_end = (char *)(leaf->items + nritems + 1);
		return (char *)(leaf->data + data_end) - (char *)items_end;
	}

```

The `leaf_data_end()` calculates the `offset` of the last data item (from the `leaf->data` base address). The `items_end` in `leaf_free_space()` is the end address of the last `struct item`. Thus `(char *)(leaf->data + data_end) - (char *)items_end` is the size of free space between last `struct item` and last data item.

If `leaf_free_space(leaf)` says there is no enough space for a new data item, then `split_leaf(root, &path, data_size)` is called to split the leaf. This splits the path's leaf into two, making sure there is at least `data_size` available for the resulting leaf level of the path. This is described in detail in the subsection **Splitting Leaf Node**. The splitting may return a new leaf node to be used for inserting data. However this still needs another verification, so it casts again to `struct leaf` pointer and checks again for proper free space. If it fails, it is surely making a `BUG()`.

Then the 3rd thing for `insert_item()` is to get a `slot` as `path.slots[0]`, which is the place to insert data into the leaf. However, if `slot` is `0` then `fixup_low_keys(&path, key, 1)` is called to adjust the pointers going up the tree, starting at `level` making sure the right key of each node points to 'key'. This is used after shifting pointers to the left, so it stops fixing up pointers when a given leaf/node is not in slot 0 of the higher levels.

```c

	static void fixup_low_keys(struct ctree_path *path, struct key *key,
				     int level)
	{
		int i;
		/* adjust the pointers going up the tree */
		for (i = level; i < MAX_LEVEL; i++) {
			struct node *t = path->nodes[i];
			int tslot = path->slots[i];
			if (!t)
				break;
			memcpy(t->keys + tslot, key, sizeof(*key));
			if (tslot != 0)
				break;
		}
	}

```

The next thing for `insert_item()` is based on the `slot` position for the inserting data item. If `slot` is not equal to `nritems`, it means inserting the data item in an existing item. In this case, all data items from `slot` to `nritems` will have to `shfit left` (being moved one `data_size` offset, thus each of these `offset` are subtracted by `data_size`). The `items` and `data` from `slot` to `nritems` are also moved by `memmove()`.

Finally, the `insert_item()` puts in place for the new data item, the `key`, `offset`, and `size` fields of `struct item` for the proper `slot` are updated, and `data` is copied in place for `data_size` bytes. The `nritems` in the header is increased by 1 indicating a complete of new data item insertion. To be sure, `leaf_free_space(leaf)` is called to verify the leaf node is not having `negtive` free space.

## Splitting Leaf Node

As we have seen in `insert_item()`, when a leaf node does not have enough space for a new data item, it tries to split the leaf node into two, and insert the data into the newly split resultant node. This is done by `split_leaf()`.

```c

	int split_leaf(struct ctree_root *root, struct ctree_path *path, int data_size)
	{
		struct leaf *l = (struct leaf *)path->nodes[0];
		int nritems = l->header.nritems;
		int mid = (nritems + 1)/ 2;
		int slot = path->slots[0];
		struct leaf *right;
		int space_needed = data_size + sizeof(struct item);
		int data_copy_size;
		int rt_data_off;
		int i;
		int ret;
	
		if (push_leaf_left(root, path, data_size) == 0) {
			return 0;
		}
		right = malloc(sizeof(struct leaf));
		memset(right, 0, sizeof(*right));
		if (mid <= slot) {
			if (leaf_space_used(l, mid, nritems - mid) + space_needed >
				LEAF_DATA_SIZE)
				BUG();
		} else {
			if (leaf_space_used(l, 0, mid + 1) + space_needed >
				LEAF_DATA_SIZE)
				BUG();
		}
		right->header.nritems = nritems - mid;
		data_copy_size = l->items[mid].offset + l->items[mid].size -
				 leaf_data_end(l);
		memcpy(right->items, l->items + mid,
		       (nritems - mid) * sizeof(struct item));
		memcpy(right->data + LEAF_DATA_SIZE - data_copy_size,
		       l->data + leaf_data_end(l), data_copy_size);
		rt_data_off = LEAF_DATA_SIZE -
			     (l->items[mid].offset + l->items[mid].size);
		for (i = 0; i < right->header.nritems; i++) {
			right->items[i].offset += rt_data_off;
		}
		l->header.nritems = mid;
		ret = insert_ptr(root, path, &right->items[0].key,
				  (u64)right, 1);
		if (mid <= slot) {
			path->nodes[0] = (struct node *)right;
			path->slots[0] -= mid;
			path->slots[1] += 1;
		}
		return ret;
	}

```

The most complicated code in this function is the `push_leaf_left()`, which pushes some data in the path leaf to the left, trying to free up at least `data_size` bytes, it returns zero if the push worked, nonzero otherwise. This is described in detail in the next subsection **Pushing Left for Items in Leaf Node**.

If the `push_leaf_left()` fails, it means a new leaf should be inserted. It allocates a `struct leaf` and marking it `right`. Then it determines if there is enough room for the new data item to insert `right` on the `slot` popsition.

## Pushing Left for Items in Leaf Node

```c

	int push_leaf_left(struct ctree_root *root, struct ctree_path *path,
			   int data_size)
	{
		struct leaf *right = (struct leaf *)path->nodes[0];
		struct leaf *left;
		int slot;
		int i;
		int free_space;
		int push_space = 0;
		int push_items = 0;
		struct item *item;
		int old_left_nritems;
	
		slot = path->slots[1];
		if (slot == 0) {
			return 1;
		}
		if (!path->nodes[1]) {
			return 1;
		}
		left = read_block(path->nodes[1]->blockptrs[slot - 1]);
		free_space = leaf_free_space(left);
		if (free_space < data_size + sizeof(struct item)) {
			return 1;
		}
		for (i = 0; i < right->header.nritems; i++) {
			item = right->items + i;
			if (path->slots[0] == i)
				push_space += data_size + sizeof(*item);
			if (item->size + sizeof(*item) + push_space > free_space)
				break;
			push_items++;
			push_space += item->size + sizeof(*item);
		}
		if (push_items == 0) {
			return 1;
		}
		/* push data from right to left */
		memcpy(left->items + left->header.nritems,
			right->items, push_items * sizeof(struct item));
		push_space = LEAF_DATA_SIZE - right->items[push_items -1].offset;
		memcpy(left->data + leaf_data_end(left) - push_space,
			right->data + right->items[push_items - 1].offset,
			push_space);
		old_left_nritems = left->header.nritems;
		for(i = old_left_nritems; i < old_left_nritems + push_items; i++) {
			left->items[i].offset -= LEAF_DATA_SIZE -
				left->items[old_left_nritems -1].offset;
		}
		left->header.nritems += push_items;
	
		/* fixup right node */
		push_space = right->items[push_items-1].offset - leaf_data_end(right);
		memmove(right->data + LEAF_DATA_SIZE - push_space, right->data +
			leaf_data_end(right), push_space);
		memmove(right->items, right->items + push_items,
			(right->header.nritems - push_items) * sizeof(struct item));
		right->header.nritems -= push_items;
		push_space = LEAF_DATA_SIZE;
		for (i = 0; i < right->header.nritems; i++) {
			right->items[i].offset = push_space - right->items[i].size;
			push_space = right->items[i].offset;
		}
		fixup_low_keys(path, &right->items[0].key, 1);
	
		/* then fixup the leaf pointer in the path */
		if (path->slots[0] < push_items) {
			path->slots[0] += old_left_nritems;
			path->nodes[0] = (struct node*)left;
			path->slots[1] -= 1;
		} else {
			path->slots[0] -= push_items;
		}
		return 0;
	}

```

The 1st step in `push_leaf_left()` is to check if the parent (level 1) of the leaf node is on slot 0, which means it is not possible to `push left` since the leaf is already on the most left side. Then the `read_block(path->nodes[1]->blockptrs[slot - 1])` reads in the node to the left of the current leaf node, and mark it `left`. And `leaf_free_space(left)` is called to check free space on this `left` node, failing the `push left` operation if there is no enough space for `data_size` (plus a `struct item`) on `left` node (the free space on the left node is saved in `push_space` variable). Then a `for` loop is performed to calculate the number of items that can be `pushed` to `left`, this starts from slot 0 on `right` leaf node, for each item scanned, it accumulates the total space occupied by these items in `push_space` variable. The `for` loop treats the slot that is `path->slots[0]` specially, which will be the `data_size` of data item to be inserted, and exit the loop once it is not enough to push more items from `right` to `left`. After that, if it is determined to be impossible to push any item, `push_leaf_left()` returns 1 indicating failing the `push`. The actual `push` operation is just some `memcpy()` to copy the items and data buffer from the `right` to the `left` node, and adjust the `left` offsets and the corresponding `nritems` header field. Then the `right` node will have to be fixed up by `memmove()` to move remaining items not pushed to `left` into items starting from slot 0, and offsets and header `nritems` updated correspondingly. Finally, the `path` is updated, with `fixup_low_keys()` for the parent `key`, and change `slots[0]` and `nodes[0]` to the left node when the `new data` is also `pushed`, to reflect the correct node bingding in the leaf.

<img src="{{ site.baseurl }}/images/2015-11-02-1/btrfs-struct-leaf-push-left.png" alt="btrfs struct leaf push left">

# Removing Items from BTree

```c

	int del_item(struct ctree_root *root, struct key *key)
	{
		int ret;
		int slot;
		struct leaf *leaf;
		struct ctree_path path;
		int doff;
		int dsize;
	
		init_path(&path);
		ret = search_slot(root, key, &path);
		if (ret != 0)
			return -1;
	
		leaf = (struct leaf *)path.nodes[0];
		slot = path.slots[0];
		doff = leaf->items[slot].offset;
		dsize = leaf->items[slot].size;
	
		if (slot != leaf->header.nritems - 1) {
			int i;
			int data_end = leaf_data_end(leaf);
			memmove(leaf->data + data_end + dsize,
				leaf->data + data_end,
				doff - data_end);
			for (i = slot + 1; i < leaf->header.nritems; i++)
				leaf->items[i].offset += dsize;
			memmove(leaf->items + slot, leaf->items + slot + 1,
				sizeof(struct item) *
				(leaf->header.nritems - slot - 1));
		}
		leaf->header.nritems -= 1;
		if (leaf->header.nritems == 0) {
			free(leaf);
			del_ptr(root, &path, 1);
		} else {
			if (slot == 0)
				fixup_low_keys(&path, &leaf->items[0].key, 1);
			if (leaf_space_used(leaf, 0, leaf->header.nritems) <
			    LEAF_DATA_SIZE / 4) {
				/* push_leaf_left fixes the path.
				 * make sure the path still points to our leaf
				 * for possible call to del_ptr below
				 */
				slot = path.slots[1];
				push_leaf_left(root, &path, 1);
				path.slots[1] = slot;
				if (leaf->header.nritems == 0) {
					free(leaf);
					del_ptr(root, &path, 1);
				}
			}
		}
		return 0;
	}

```

