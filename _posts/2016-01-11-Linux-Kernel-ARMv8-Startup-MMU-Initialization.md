---
layout: post
title: "Linux Kernel ARMv8 Startup MMU Initialization"
description: 
headline: 
modified: 2016-01-11
category: KernelDev
tags: [KernelDev]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

The AArch64 architecture allows up to 4 levels of translation tables with a 4KB page size and up to 3 levels with a 64KB page size. AArch64 Linux uses either 3 levels or 4 levels of translation tables with the 4KB page configuration, allowing 39-bit (512GB) or 48-bit (256TB) virtual addresses, respectively, for both user and kernel. With 64KB pages, only 2 levels of translation tables, allowing 42-bit (4TB) virtual address, are used but the memory layout is the same. This blog post describes the virtual memory layout used by the AArch64
Linux kernel as well as its implementation in its start up procedure.

## Virtual Memory Layout

According to `linux-4.2/Documentation/arm64/memory.txt`, user space addresses have bits `63:48` set to 0 while the kernel addresses have the same bits set to 1. `TTBRx` selection is given by bit 63 of the virtual address. The `swapper_pg_dir` contains only kernel (global) mappings while the user `pgd` contains only user (non-global) mappings. The `swapper_pg_dir` address is written to `TTBR1` and never written to `TTBR0`.

```c

	AArch64 Linux memory layout with 4KB pages + 3 levels:
	
	Start			End			Size		Use
	-----------------------------------------------------------------------
	0000000000000000	0000007fffffffff	 512GB		user
	ffffff8000000000	ffffffffffffffff	 512GB		kernel
	
	
	AArch64 Linux memory layout with 4KB pages + 4 levels:
	
	Start			End			Size		Use
	-----------------------------------------------------------------------
	0000000000000000	0000ffffffffffff	 256TB		user
	ffff000000000000	ffffffffffffffff	 256TB		kernel
	
	
	AArch64 Linux memory layout with 64KB pages + 2 levels:
	
	Start			End			Size		Use
	-----------------------------------------------------------------------
	0000000000000000	000003ffffffffff	   4TB		user
	fffffc0000000000	ffffffffffffffff	   4TB		kernel
	
	
	AArch64 Linux memory layout with 64KB pages + 3 levels:
	
	Start			End			Size		Use
	-----------------------------------------------------------------------
	0000000000000000	0000ffffffffffff	 256TB		user
	ffff000000000000	ffffffffffffffff	 256TB		kernel
	
	
	For details of the virtual kernel memory layout please see the kernel
	booting log.
	
	
	Translation table lookup with 4KB pages:
	
	+--------+--------+--------+--------+--------+--------+--------+--------+
	|63    56|55    48|47    40|39    32|31    24|23    16|15     8|7      0|
	+--------+--------+--------+--------+--------+--------+--------+--------+
	 |                 |         |         |         |         |
	 |                 |         |         |         |         v
	 |                 |         |         |         |   [11:0]  in-page offset
	 |                 |         |         |         +-> [20:12] L3 index
	 |                 |         |         +-----------> [29:21] L2 index
	 |                 |         +---------------------> [38:30] L1 index
	 |                 +-------------------------------> [47:39] L0 index
	 +-------------------------------------------------> [63] TTBR0/1
	
	
	Translation table lookup with 64KB pages:
	
	+--------+--------+--------+--------+--------+--------+--------+--------+
	|63    56|55    48|47    40|39    32|31    24|23    16|15     8|7      0|
	+--------+--------+--------+--------+--------+--------+--------+--------+
	 |                 |    |               |              |
	 |                 |    |               |              v
	 |                 |    |               |            [15:0]  in-page offset
	 |                 |    |               +----------> [28:16] L3 index
	 |                 |    +--------------------------> [41:29] L2 index
	 |                 +-------------------------------> [47:42] L1 index
	 +-------------------------------------------------> [63] TTBR0/1
	
	
	When using KVM, the hypervisor maps kernel pages in EL2, at a fixed
	offset from the kernel VA (top 24bits of the kernel VA set to zero):
	
	Start			End			Size		Use
	-----------------------------------------------------------------------
	0000004000000000	0000007fffffffff	 256GB		kernel objects mapped in HYP

```

## Before Initial Page Table Creation

The `linux-4.2/arch/arm64/kernel/head.S` has some header handling in its first lines of code, then it follows the `ENTRY(stext)` which is jumped to by a `b stext` at the start of the code.

```c

	ENTRY(stext)
		bl	preserve_boot_args
		bl	el2_setup			// Drop to EL1, w20=cpu_boot_mode
		adrp	x24, __PHYS_OFFSET
		bl	set_cpu_boot_mode_flag
		bl	__create_page_tables		// x25=TTBR0, x26=TTBR1
		/*
		 * The following calls CPU setup code, see arch/arm64/mm/proc.S for
		 * details.
		 * On return, the CPU will be ready for the MMU to be turned on and
		 * the TCR will have been set.
		 */
		ldr	x27, =__mmap_switched		// address to jump to after
							// MMU has been enabled
		adr_l	lr, __enable_mmu		// return (PIC) address
		b	__cpu_setup			// initialise processor
	ENDPROC(stext)

```

The `preserve_boot_args()` preserves the arguments passed by the bootloader in `x0 .. x3` to `boot_args` array, defined in `linux-4.2/arch/arm64/kernel/setup.c`.

```c

	/*
	 * The recorded values of x0 .. x3 upon kernel entry.
	 */
	u64 __cacheline_aligned boot_args[4];

```

The `el2_setup()` tries to setup the processor to run in EL1 exception level if the processor was entered in EL2. It returns either BOOT_CPU_MODE_EL1 or BOOT_CPU_MODE_EL2 in `x20` if booted in EL1 or EL2 respectively. The registers setup in this routine are as following:

* `hcr_el2` set to `(1 << 31)` so `RW, bit [31]` is set which makes sure `64-bit EL1` (The Execution state for EL1 is AArch64. The Execution state for EL0 is determined by the current value of `PSTATE.nRW` when executing at EL0).
* `cnthctl_el2` set to `enables EL1 physical timers`, with `EL1PCEN, bit [1]` (Traps Non-secure EL0 and EL1 accesses to the physical timer registers to EL2) and `EL1PCTEN, bit [0]` (Traps Non-secure EL0 and EL1 accesses to the physical counter register to EL2) both set.
* `cntvoff_el2` set to 0 so it `clears virtual offset`.
* `ICC_SRE_EL2` set to make `ICC_SRE_EL2.SRE==1` and `ICC_SRE_EL2.Enable==1`, thus `makes sure SRE is now set`. The `ICC_SRE_EL2, Interrupt Controller System Register Enable register (EL2)` controls whether the `System register interface` or the `memory-mapped interface` to the GIC CPU interface is used for EL2. The `SRE, bit [0]` set to 1 makes the `the System register interface to the ICH_* registers and the EL1 and EL2 ICC_* registers is enabled for EL2`.
* `ICH_HCR_EL2` set to 0 so it `resets ICC_HCR_EL2 to defaults`. The `ICH_HCR_EL2, Interrupt Controller Hyp Control Register` controls the environment for VMs.
* `midr_el1` and `mpidr_el1` are copied to `vpidr_el2` (Holds the value of the `Virtualization Processor ID`. This is the value returned by Non-secure EL1 reads of `MIDR_EL1`) and `vmpidr_el2` (Holds the value of the `Virtualization Multiprocessor ID`. This is the value returned by Non-secure EL1 reads of `MPIDR_EL1`) respectively. This copying makes the IDs not changed as seen from either EL1 or EL2.
* `sctlr_el1` is set to `Set EE and E0E on BE systems` or `Clear EE and E0E on LE systems`. The `EE, bit [25]` in `SCTLR_EL1, System Control Register (EL1)` controls `endianness of data accesses at EL1, and stage 1 translation table walks in the EL1&0 translation regime`, and `E0E, bit [24]` controls `Endianness of data accesses at EL0`.
* `cptr_el2` and `hstr_el2` are set to disable Coprocessor/CP15 traps to EL2.
* `vttbr_el2` is set to 0. The `VTTBR_EL2, Virtualization Translation Table Base Register` holds the base address of the translation table for the `stage 2 translation` of memory accesses from Non-secure EL0 and EL1.
* `vbar_el2` is set to `__hyp_stub_vectors`, which holds the vector base address for any exception that is taken to EL2.
* `spsr_el2` is set to `(PSR_F_BIT | PSR_I_BIT | PSR_A_BIT | PSR_D_BIT | PSR_MODE_EL1h)`.
* `elr_el2` is set to the `lr` that the calling code is to return to, so that after the `eret` instruction the processor returns to the caller of `el2_setup()`.

```c

	/*
	 * If we're fortunate enough to boot at EL2, ensure that the world is
	 * sane before dropping to EL1.
	 *
	 * Returns either BOOT_CPU_MODE_EL1 or BOOT_CPU_MODE_EL2 in x20 if
	 * booted in EL1 or EL2 respectively.
	 */
	ENTRY(el2_setup)
		mrs	x0, CurrentEL
		cmp	x0, #CurrentEL_EL2
		b.ne	1f
		mrs	x0, sctlr_el2
	CPU_BE(	orr	x0, x0, #(1 << 25)	)	// Set the EE bit for EL2
	CPU_LE(	bic	x0, x0, #(1 << 25)	)	// Clear the EE bit for EL2
		msr	sctlr_el2, x0
		b	2f
	1:	mrs	x0, sctlr_el1
	CPU_BE(	orr	x0, x0, #(3 << 24)	)	// Set the EE and E0E bits for EL1
	CPU_LE(	bic	x0, x0, #(3 << 24)	)	// Clear the EE and E0E bits for EL1
		msr	sctlr_el1, x0
		mov	w20, #BOOT_CPU_MODE_EL1		// This cpu booted in EL1
		isb
		ret
	
		/* Hyp configuration. */
	2:	mov	x0, #(1 << 31)			// 64-bit EL1
		msr	hcr_el2, x0
	
		/* Generic timers. */
		mrs	x0, cnthctl_el2
		orr	x0, x0, #3			// Enable EL1 physical timers
		msr	cnthctl_el2, x0
		msr	cntvoff_el2, xzr		// Clear virtual offset
	
	#ifdef CONFIG_ARM_GIC_V3
		/* GICv3 system register access */
		mrs	x0, id_aa64pfr0_el1
		ubfx	x0, x0, #24, #4
		cmp	x0, #1
		b.ne	3f
	
		mrs_s	x0, ICC_SRE_EL2
		orr	x0, x0, #ICC_SRE_EL2_SRE	// Set ICC_SRE_EL2.SRE==1
		orr	x0, x0, #ICC_SRE_EL2_ENABLE	// Set ICC_SRE_EL2.Enable==1
		msr_s	ICC_SRE_EL2, x0
		isb					// Make sure SRE is now set
		msr_s	ICH_HCR_EL2, xzr		// Reset ICC_HCR_EL2 to defaults
	
	3:
	#endif
	
		/* Populate ID registers. */
		mrs	x0, midr_el1
		mrs	x1, mpidr_el1
		msr	vpidr_el2, x0
		msr	vmpidr_el2, x1
	
		/* sctlr_el1 */
		mov	x0, #0x0800			// Set/clear RES{1,0} bits
	CPU_BE(	movk	x0, #0x33d0, lsl #16	)	// Set EE and E0E on BE systems
	CPU_LE(	movk	x0, #0x30d0, lsl #16	)	// Clear EE and E0E on LE systems
		msr	sctlr_el1, x0
	
		/* Coprocessor traps. */
		mov	x0, #0x33ff
		msr	cptr_el2, x0			// Disable copro. traps to EL2
	
	#ifdef CONFIG_COMPAT
		msr	hstr_el2, xzr			// Disable CP15 traps to EL2
	#endif
	
		/* Stage-2 translation */
		msr	vttbr_el2, xzr
	
		/* Hypervisor stub */
		adrp	x0, __hyp_stub_vectors
		add	x0, x0, #:lo12:__hyp_stub_vectors
		msr	vbar_el2, x0
	
		/* spsr */
		mov	x0, #(PSR_F_BIT | PSR_I_BIT | PSR_A_BIT | PSR_D_BIT |\
			      PSR_MODE_EL1h)
		msr	spsr_el2, x0
		msr	elr_el2, lr
		mov	w20, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
		eret
	ENDPROC(el2_setup)

```

Then `set_cpu_boot_mode_flag()` sets the `__boot_cpu_mode flag` depending on the CPU boot mode passed in `x20`.

```c

	/*
	 * Sets the __boot_cpu_mode flag depending on the CPU boot mode passed
	 * in x20. See arch/arm64/include/asm/virt.h for more info.
	 */
	ENTRY(set_cpu_boot_mode_flag)
		adr_l	x1, __boot_cpu_mode
		cmp	w20, #BOOT_CPU_MODE_EL2
		b.ne	1f
		add	x1, x1, #4
	1:	str	w20, [x1]			// This CPU has booted in EL1
		dmb	sy
		dc	ivac, x1			// Invalidate potentially stale cache line
		ret
	ENDPROC(set_cpu_boot_mode_flag)

```

## Creating Page Tables

The code that kickstarts setting up of MMU is `__create_page_tables()` in `linux-4.2/arch/arm64/kernel/head.S`.

```c

	/*
	 * Setup the initial page tables. We only setup the barest amount which is
	 * required to get the kernel running. The following sections are required:
	 *   - identity mapping to enable the MMU (low address, TTBR0)
	 *   - first few MB of the kernel linear mapping to jump to once the MMU has
	 *     been enabled
	 */
	__create_page_tables:
		adrp	x25, idmap_pg_dir
		adrp	x26, swapper_pg_dir
		mov	x27, lr
	
		/*
		 * Invalidate the idmap and swapper page tables to avoid potential
		 * dirty cache lines being evicted.
		 */
		mov	x0, x25
		add	x1, x26, #SWAPPER_DIR_SIZE
		bl	__inval_cache_range
	
		/*
		 * Clear the idmap and swapper page tables.
		 */
		mov	x0, x25
		add	x6, x26, #SWAPPER_DIR_SIZE
	1:	stp	xzr, xzr, [x0], #16
		stp	xzr, xzr, [x0], #16
		stp	xzr, xzr, [x0], #16
		stp	xzr, xzr, [x0], #16
		cmp	x0, x6
		b.lo	1b
	
		ldr	x7, =MM_MMUFLAGS
	
		/*
		 * Create the identity mapping.
		 */
		mov	x0, x25				// idmap_pg_dir
		adrp	x3, __idmap_text_start		// __pa(__idmap_text_start)
	
	#ifndef CONFIG_ARM64_VA_BITS_48
	#define EXTRA_SHIFT	(PGDIR_SHIFT + PAGE_SHIFT - 3)
	#define EXTRA_PTRS	(1 << (48 - EXTRA_SHIFT))
	
		/*
		 * If VA_BITS < 48, it may be too small to allow for an ID mapping to be
		 * created that covers system RAM if that is located sufficiently high
		 * in the physical address space. So for the ID map, use an extended
		 * virtual range in that case, by configuring an additional translation
		 * level.
		 * First, we have to verify our assumption that the current value of
		 * VA_BITS was chosen such that all translation levels are fully
		 * utilised, and that lowering T0SZ will always result in an additional
		 * translation level to be configured.
		 */
	#if VA_BITS != EXTRA_SHIFT
	#error "Mismatch between VA_BITS and page size/number of translation levels"
	#endif
	
		/*
		 * Calculate the maximum allowed value for TCR_EL1.T0SZ so that the
		 * entire ID map region can be mapped. As T0SZ == (64 - #bits used),
		 * this number conveniently equals the number of leading zeroes in
		 * the physical address of __idmap_text_end.
		 */
		adrp	x5, __idmap_text_end
		clz	x5, x5
		cmp	x5, TCR_T0SZ(VA_BITS)	// default T0SZ small enough?
		b.ge	1f			// .. then skip additional level
	
		adr_l	x6, idmap_t0sz
		str	x5, [x6]
		dmb	sy
		dc	ivac, x6		// Invalidate potentially stale cache line
	
		create_table_entry x0, x3, EXTRA_SHIFT, EXTRA_PTRS, x5, x6
	1:
	#endif
	
		create_pgd_entry x0, x3, x5, x6
		mov	x5, x3				// __pa(__idmap_text_start)
		adr_l	x6, __idmap_text_end		// __pa(__idmap_text_end)
		create_block_map x0, x7, x3, x5, x6
	
		/*
		 * Map the kernel image (starting with PHYS_OFFSET).
		 */
		mov	x0, x26				// swapper_pg_dir
		mov	x5, #PAGE_OFFSET
		create_pgd_entry x0, x5, x3, x6
		ldr	x6, =KERNEL_END			// __va(KERNEL_END)
		mov	x3, x24				// phys offset
		create_block_map x0, x7, x3, x5, x6
	
		/*
		 * Since the page tables have been populated with non-cacheable
		 * accesses (MMU disabled), invalidate the idmap and swapper page
		 * tables again to remove any speculatively loaded cache lines.
		 */
		mov	x0, x25
		add	x1, x26, #SWAPPER_DIR_SIZE
		dmb	sy
		bl	__inval_cache_range
	
		mov	lr, x27
		ret
	ENDPROC(__create_page_tables)

```

### Perform Identity Mapping

There is a git commit `ARM: idmap: populate identity map pgd at init time using .init.text` which has the following comments:

> When disabling and re-enabling the MMU, it is necessary to take out an identity mapping for the code that manipulates the `SCTLR` in order to avoid it disappearing from under our feet. This is useful when soft rebooting and returning from CPU suspend.

> This patch allocates a set of page tables during boot and populates them with an identity mapping for the `.idmap.text` section. This means that users of the identity map do not need to manage their own pgd and can instead annotate their functions with `__idmap` or, in the case of assembly code, place them in the correct section.

To understand why indentify mapping is required, lets say before mmu is turned on PC is at physical address XXX, also say at XXX there is going to be mmu on instruction. The next instruction fetch would be from XXX + 4 which would now be VIRTUAL address (as mmu is turned on), which should still get converted to (via page tables set up as above) to XXX + 4 (in physical). This is called identity mapping as specified in the comments. When the kernel starts, the MMU is off, and ther ARM is running with an implicit identity mapping (i.e. each virtual address maps to the same physical address). If your physical memory starts at 0x80000000, then the PC will be 0x800xxxxx. When the MMU table is turned on, the PC is still at 0x800xxxx, so even though the kernel has 0xc00xxxxx mapped to 0x800xxxxx it also has to have 0x800xxxxx mapped to 0x800xxxxx. So this mapping of 0x800xxxxx to 0x800xxxxx is the "identity" portion and is needed while switching the MMU on. The 0xc00xxxxx to 0x800xxxxx mapping is what's used while the kernel is running. For 64 bit systems, the 0xc00xxxxx would be something like 0xfffffcxxxxxxxxxx.

The `IDMAP_TEXT`, `idmap_pg_dir` and `swapper_pg_dir` are defined in `linux-4.2/arch/arm64/kernel/vmlinux.lds.S`.

```c

	SECTIONS {
		. = PAGE_OFFSET + TEXT_OFFSET;
	
		.head.text : {
			_text = .;
			HEAD_TEXT
		}
	...
		.text : {			/* Real text segment		*/
			_stext = .;		/* Text and read-only data	*/
				__exception_text_start = .;
				*(.exception.text)
				__exception_text_end = .;
				IRQENTRY_TEXT
				TEXT_TEXT
				SCHED_TEXT
				LOCK_TEXT
				HYPERVISOR_TEXT
				IDMAP_TEXT
				*(.fixup)
				*(.gnu.warning)
			. = ALIGN(16);
			*(.got)			/* Global offset table		*/
		}

		BSS_SECTION(0, 0, 0)

		. = ALIGN(PAGE_SIZE);
		idmap_pg_dir = .;
		. += IDMAP_DIR_SIZE;
		swapper_pg_dir = .;
		. += SWAPPER_DIR_SIZE;
	...
	}

```

The `TEXT_OFFSET` is generated in `linux-4.2/arch/arm64/Makefile`.

```c

	# The byte offset of the kernel image in RAM from the start of RAM.
	ifeq ($(CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET), y)
	TEXT_OFFSET := $(shell awk 'BEGIN {srand(); printf "0x%03x000\n", int(512 * rand())}')
	else
	TEXT_OFFSET := 0x00080000
	endif

```

The `IDMAP_DIR_SIZE` is defined in `linux-4.2/arch/arm64/include/asm/page.h`.

```c

	/*
	 * The idmap and swapper page tables need some space reserved in the kernel
	 * image. Both require pgd, pud (4 levels only) and pmd tables to (section)
	 * map the kernel. With the 64K page configuration, swapper and idmap need to
	 * map to pte level. The swapper also maps the FDT (see __create_page_tables
	 * for more information). Note that the number of ID map translation levels
	 * could be increased on the fly if system RAM is out of reach for the default
	 * VA range, so 3 pages are reserved in all cases.
	 */
	#ifdef CONFIG_ARM64_64K_PAGES
	#define SWAPPER_PGTABLE_LEVELS	(CONFIG_PGTABLE_LEVELS)
	#else
	#define SWAPPER_PGTABLE_LEVELS	(CONFIG_PGTABLE_LEVELS - 1)
	#endif
	
	#define SWAPPER_DIR_SIZE	(SWAPPER_PGTABLE_LEVELS * PAGE_SIZE)
	#define IDMAP_DIR_SIZE		(3 * PAGE_SIZE)

```

The `__create_page_tables()` starts by setting addresses of `idmap_pg_dir` and `swapper_pg_dir` into `x25` and `x26`, then follows by calling  `__inval_cache_range(start, end)` to invalidate the idmap and swapper page tables to avoid potential dirty cache lines being evicted. It further follows with a bunch of code to clear the idmap and swapper page tables. I think the clearing of the idmap and swapper page tables should be performed before invalidating these page tables, because the clearing code would definitely bring the memory into the caches, thus voiding the effects of the invalidation.

The `IDMAP_TEXT` in the `linux-4.2/arch/arm64/kernel/vmlinux.lds.S` is defined as below, which exports the `__idmap_text_start` (note the `ALIGN(SZ_4K)`) and `__idmap_text_end`.

```c

	#define IDMAP_TEXT					\
		. = ALIGN(SZ_4K);				\
		VMLINUX_SYMBOL(__idmap_text_start) = .;		\
		*(.idmap.text)					\
		VMLINUX_SYMBOL(__idmap_text_end) = .;

```

With this, in the `__create_page_tables()`, after it invalidates and clears the `idmap_pg_dir`, it tries to map the [`__idmap_text_start .. __idmap_text_end`] area with this `idmap_pg_dir` page tables. 

If `CONFIG_ARM64_VA_BITS_48` is not configured, which should be mostly default with either `ARM64_4K_PAGES` or `ARM64_64K_PAGES`, then the `__create_page_tables()` would try to configure an additional translation level if it is too small to allow for an ID mapping to be created that covers system RAM if that is located sufficiently high in the physical address space.

```c

	choice
		prompt "Virtual address space size"
		default ARM64_VA_BITS_39 if ARM64_4K_PAGES
		default ARM64_VA_BITS_42 if ARM64_64K_PAGES
		help
		  Allows choosing one of multiple possible virtual address
		  space sizes. The level of translation table is determined by
		  a combination of page size and virtual address space size.
	
	config ARM64_VA_BITS_39
		bool "39-bit"
		depends on ARM64_4K_PAGES
	
	config ARM64_VA_BITS_42
		bool "42-bit"
		depends on ARM64_64K_PAGES
	
	config ARM64_VA_BITS_48
		bool "48-bit"
	
	endchoice

```

We check this call `create_table_entry x0, x3, EXTRA_SHIFT, EXTRA_PTRS, x5, x6`, with the parameters described as below.

* `x0` is the virtual address of `idmap_pg_dir`.
* `x3` is physical address of `__idmap_text_start`, that is `__pa(__idmap_text_start)`.
* `EXTRA_SHIFT` is defined as `(PGDIR_SHIFT + PAGE_SHIFT - 3)`
* `EXTRA_PTRS` is defined as `(1 << (48 - EXTRA_SHIFT))`.
* `x5` is the maximum allowed value for `TCR_EL1.T0SZ` so that the entire ID map region can be mapped. As `T0SZ == (64 - #bits used)`, this number conveniently equals the number of leading zeroes in the physical address of `__idmap_text_end`.
* `x6` is the address of variable `u64 idmap_t0sz = TCR_T0SZ(VA_BITS);` as defined in `linux-4.2/arch/arm64/mm/mmu.c`, the value of the variable is updated to be `x5`.

```c

	/*
	 * Macro to create a table entry to the next page.
	 *
	 *	tbl:	page table address
	 *	virt:	virtual address
	 *	shift:	#imm page table shift
	 *	ptrs:	#imm pointers per table page
	 *
	 * Preserves:	virt
	 * Corrupts:	tmp1, tmp2
	 * Returns:	tbl -> next level table page address
	 */
		.macro	create_table_entry, tbl, virt, shift, ptrs, tmp1, tmp2
		lsr	\tmp1, \virt, #\shift
		and	\tmp1, \tmp1, #\ptrs - 1	// table index
		add	\tmp2, \tbl, #PAGE_SIZE
		orr	\tmp2, \tmp2, #PMD_TYPE_TABLE	// address of next table and entry type
		str	\tmp2, [\tbl, \tmp1, lsl #3]
		add	\tbl, \tbl, #PAGE_SIZE		// next level table page
		.endm

```

According to the macro above, it does the following:

* Calculates an `table index` in the `idmap_pg_dir` according to the `shift` parameter (`EXTRA_SHIFT` in this call).
* Generate the virtual address of next page in `tbl` (`idmap_pg_dir` in this call).
* Set the entry as specified by `table index` in `tbl` (`idmap_pg_dir` in this call) to point to the next page in `tbl` (`idmap_pg_dir` in this call).
* Seth that entry to be `PMD_TYPE_TABLE`.
* Make `tbl` register (`x0` in this call) to point to the next page in `tbl` (`idmap_pg_dir` in this call).

Then we follow to `create_pgd_entry x0, x3, x5, x6`, with the parameters described as below.

* `x0` is the virtual address of `idmap_pg_dir`.
* `x3` is physical address of `__idmap_text_start`, that is `__pa(__idmap_text_start)`.
* `x5` is used as 1st temporary register.
* `x6` is used as 2nd temporary register.

```c

	/*
	 * Macro to populate the PGD (and possibily PUD) for the corresponding
	 * block entry in the next level (tbl) for the given virtual address.
	 *
	 * Preserves:	tbl, next, virt
	 * Corrupts:	tmp1, tmp2
	 */
		.macro	create_pgd_entry, tbl, virt, tmp1, tmp2
		create_table_entry \tbl, \virt, PGDIR_SHIFT, PTRS_PER_PGD, \tmp1, \tmp2
	#if SWAPPER_PGTABLE_LEVELS == 3
		create_table_entry \tbl, \virt, TABLE_SHIFT, PTRS_PER_PTE, \tmp1, \tmp2
	#endif
		.endm

```

Note that in the previous calls to `create_table_entry()`, the `virt` parameter is actually passed as `physical address` of `__idmap_text_start`, this is intentional since that is why we are doing `identity mapping` for the [`__idmap_text_start .. __idmap_text_end`] area.

We then get to the `create_block_map x0, x7, x3, x5, x6` macro call, with the following parameters:

* `x0` is the lowest level of the `idmap_pg_dir` after the two or three `create_table_entry()` calls.
* `x7` is loaded with `ldr	x7, =MM_MMUFLAGS` so it is something like `PMD_ATTRINDX(MT_NORMAL) | PMD_FLAGS`.
* `x3` is physical address of `__idmap_text_start`, that is `__pa(__idmap_text_start)`.
* `x5` is the same as `x3` so it is also physical address of `__idmap_text_start`.
* `x6` is physical address of `__idmap_text_end`, that is `__pa(__idmap_text_end)`.

```c

	/*
	 * Macro to populate block entries in the page table for the start..end
	 * virtual range (inclusive).
	 *
	 * Preserves:	tbl, flags
	 * Corrupts:	phys, start, end, pstate
	 */
		.macro	create_block_map, tbl, flags, phys, start, end
		lsr	\phys, \phys, #BLOCK_SHIFT
		lsr	\start, \start, #BLOCK_SHIFT
		and	\start, \start, #PTRS_PER_PTE - 1	// table index
		orr	\phys, \flags, \phys, lsl #BLOCK_SHIFT	// table entry
		lsr	\end, \end, #BLOCK_SHIFT
		and	\end, \end, #PTRS_PER_PTE - 1		// table end index
	9999:	str	\phys, [\tbl, \start, lsl #3]		// store the entry
		add	\start, \start, #1			// next entry
		add	\phys, \phys, #BLOCK_SIZE		// next block
		cmp	\start, \end
		b.ls	9999b
		.endm

```

* Generate initial `phys` with `flags` set in place.
* Calculate the `table index` in `start`.
* Calculate the end of the `table index` for the mapping.
* Loop to store the `phys` with `flags`, increase each time with `BLOCK_SIZE`.

The corresponding definitions are as bellow.

```c

	#ifdef CONFIG_ARM64_64K_PAGES
	#define BLOCK_SHIFT	PAGE_SHIFT
	#define BLOCK_SIZE	PAGE_SIZE
	#define TABLE_SHIFT	PMD_SHIFT
	#else
	#define BLOCK_SHIFT	SECTION_SHIFT
	#define BLOCK_SIZE	SECTION_SIZE
	#define TABLE_SHIFT	PUD_SHIFT
	#endif

	#define KERNEL_START	_text
	#define KERNEL_END	_end

	/*
	 * Initial memory map attributes.
	 */
	#ifndef CONFIG_SMP
	#define PTE_FLAGS	PTE_TYPE_PAGE | PTE_AF
	#define PMD_FLAGS	PMD_TYPE_SECT | PMD_SECT_AF
	#else
	#define PTE_FLAGS	PTE_TYPE_PAGE | PTE_AF | PTE_SHARED
	#define PMD_FLAGS	PMD_TYPE_SECT | PMD_SECT_AF | PMD_SECT_S
	#endif
	
	#ifdef CONFIG_ARM64_64K_PAGES
	#define MM_MMUFLAGS	PTE_ATTRINDX(MT_NORMAL) | PTE_FLAGS
	#else
	#define MM_MMUFLAGS	PMD_ATTRINDX(MT_NORMAL) | PMD_FLAGS
	#endif

```

### Perform Kernel Mapping

There are the kernel `swapper_pg_dir` mapping initialization as below.

```c

		/*
		 * Map the kernel image (starting with PHYS_OFFSET).
		 */
		mov	x0, x26				// swapper_pg_dir
		mov	x5, #PAGE_OFFSET
		create_pgd_entry x0, x5, x3, x6
		ldr	x6, =KERNEL_END			// __va(KERNEL_END)
		mov	x3, x24				// phys offset
		create_block_map x0, x7, x3, x5, x6

```

The `create_pgd_entry x0, x5, x3, x6` has the parameters described as below.

* `x0` is the virtual address of `swapper_pg_dir`.
* `x5` is the virtual address at `PAGE_OFFSET` which is defined as `(UL(0xffffffffffffffff) << (VA_BITS - 1))` in `linux-4.2/arch/arm64/include/asm/memory.h`.
* `x3` is used as 1st temporary register.
* `x6` is used as 2nd temporary register.

The `create_block_map x0, x7, x3, x5, x6` has the parameters described as below.

* `x0` is the lowest level of the `swapper_pg_dir` after the two `create_table_entry()` calls done in  `create_pgd_entry()`.
* `x7` is loaded with `ldr	x7, =MM_MMUFLAGS` so it is something like `PMD_ATTRINDX(MT_NORMAL) | PMD_FLAGS`.
* `x3` is loaded from `x24`, which was loaded by `adrp	x24, __PHYS_OFFSET`, as physical offset of value `(KERNEL_START - TEXT_OFFSET)`.
* `x5` is the same as `x3` so it is also physical offset of value `(KERNEL_START - TEXT_OFFSET)`.
* `x6` is physical address of `KERNEL_END`.

This creates the mapping beteween virtual address `PAGE_OFFSET` to the physical address `KERNEL_START`.

## References

This blog entry refers code that is from Linux Kernel 4.2 release.

