---
layout: post
title: "TBoot - Trusted Boot with Intel TXT - Booting Stage"
description:
headline:
modified: 2015-10-13
category: SecureDev
tags: [Security]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

The primary goal of using Intel TXT is to validate that there have been no unauthorized changes to critical parts of the code that provides the secure environment. This check is performed each time the environment launches, whether it is a cold boot, warm boot, or exiting one hypervisor and launching a new one. This blog entry is the 1st in a series of blogs trying to understand TBoot, and will focus on the inital booting stage.

# Intel TXT Introduction

The Intel TXT technology supports both a static chain of trust and a dynamic chain of trust. The static chain of trust starts when the platform powers on (or the platform is reset), which resets all PCRs to their default value. For server platforms, the first measurement is made by hardware (i.e., the processor) to measure a digitally signed module (called an Authenticated Code Module or ACM) provided by the chipset manufacturer. The processor validates the signature and integrity of the signed module before executing it. The ACM then measures the first BIOS code module, which can make additional measurements.

The measurements of the ACM and BIOS code modules are extended to PCR0, which is said to hold the static core root of trust measurement (CRTM) as well as the measurement of the BIOS Trusted Computing Base (TCB). The BIOS measures additional components into PCRs as follows:

- PCR0 – CRTM, BIOS code, and Host Platform Extensions
- PCR1 – Host Platform Configuration
- PCR2 – Option ROM Code
- PCR3 – Option ROM Configuration and Data
- PCR4 – IPL (Initial Program Loader) Code (usually the Master Boot Record – MBR)
- PCR5 – IPL Code Configuration and Data (for use by the IPL Code)
- PCR6 – State Transition and Wake Events
- PCR7 – Host Platform Manufacturer Control

The dynamic chain of trust starts when the operating system invokes a special security instruction, which resets dynamic PCRs (PCR17-22) to their default value and starts the measured launch. The first dynamic measurement is made by hardware (i.e., the processor) to measure another digitally signed module (referred to as the SINIT ACM) which is also provided by the chipset manufacturer and whose signature and integrity are verified by the processor. This is known as the Dynamic Root of Trust Measurement (DRTM).

The SINIT ACM then measures the first operating system code module (referred to as the Measured Launch Environment – MLE). Before the MLE is allowed to execute, the SINIT ACM verifies that the platform meets the requirements of the Launch Control Policy (LCP) set by the platform owner. LCP consists of three parts:

1. Verifying that the SINIT version is equal or newer than the value specified
2. Verifying that the platform configuration (PCONF) is valid by comparing PCR0-7 to known-good values (the platform owner decides which PCRs to include)
3. Verifying that the MLE is valid, by comparing its measurement to a list of known-good measurements.

The integrity of the LCP and its lists of known-good measurements are protected by storing a hash measurement of the policy in the TPM in a protected non-volatile location that can only be modified by the platform owner.

The following is the `Measured Launch Phases` of Intel TXT.

<img src="{{ site.url }}/images/2015-10-13-1/txt-timeline.png" alt="TXT Measured Launch Phases">

TBOOT is regarded as Measured Launch Environment (MLE) in the image above.

# TBoot Memory Layout

The linker script file is based on `tboot-1.8.3/tboot/common/tboot.lds.x`, preprocessed with the following command in `tboot-1.8.3/tboot/Makefile`:

```console

	TARGET_LDS := $(CURDIR)/common/tboot.lds
	
	$(TARGET).gz : $(TARGET)
		gzip -f -9 < $< > $@
	
	$(TARGET) : $(OBJS) $(TARGET_LDS)
		$(LD) $(LDFLAGS) -T $(TARGET_LDS) -N $(OBJS) -o $(@D)/.$(@F).0
		$(NM) -n $(@D)/.$(@F).0 >$(TARGET)-syms
		$(LD) $(LDFLAGS) -T $(TARGET_LDS) $(LDFLAGS_STRIP) $(@D)/.$(@F).0 -o $(TARGET)
		rm -f $(@D)/.$(@F).0
	
	$(TARGET_LDS) : $(TARGET_LDS).x $(HDRS)
		$(CPP) -P -E -Ui386 $(AFLAGS) -o $@ $<
	
	$(TARGET_LDS).x : FORCE

```
The following `tboot-1.8.3/tboot/common/tboot.lds.x` contents exhibits the memory layout of the TBOOT image.

```c

	OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
	OUTPUT_ARCH(i386)
	ENTRY(start)
	PHDRS
	{
	  text PT_LOAD ;
	}
	SECTIONS
	{
	  . = TBOOT_BASE_ADDR;		/* 0x800000 */
	
	  .text : {
		*(.tboot_multiboot_header)
	  . = ALIGN(4096);
		*(.mlept)
	
	  _stext = .;	                /* text */
	  _mle_start = .;               /* beginning of MLE pages */
	
		*(.text)
		*(.fixup)
		*(.gnu.warning)
		} :text = 0x9090
	
	  _etext = .;			/* end of text section */
	
	  .rodata : { *(.rodata) *(.rodata.*) }
	  . = ALIGN(4096);
	
	  _mle_end = .;                 /* end of MLE pages */
	
	  .data : {			/* Data */
		*(.data)
		*(.tboot_shared)
		CONSTRUCTORS
		}
	
	  . = ALIGN(4096);
	
	  __bss_start = .;		/* BSS */
	  .bss : {
		*(.bss.stack_aligned)
		*(.bss.page_aligned)
		*(.bss)
		}
	
	  _end = . ;
	}

```

<img src="{{ site.url }}/images/2015-10-13-1/tboot-memory-layout.png" alt="TBoot Memory Layout">

TBoot is assuming the traditional OS Kerenel role with respect to GRUB (thus the tboot Multiboot Header is present in its 1st page of the TBoot image). The tboot module must be added as the 'kernel' in the grub.conf file. The existing 'kernel' entry should follow as a 'module'. The SINIT AC module must be added to the grub.conf boot config as the last module, e.g.:

```c

       title Xen w/ Intel(R) Trusted Execution Technology
           root (hd0,1)
           kernel /tboot.gz logging=serial,vga,memory
           module /xen.gz iommu=required dom0_mem=524288 com1=115200,8n1
           module /vmlinuz-2.6.18-xen root=/dev/VolGroup00/LogVol00 ro
           module /initrd-2.6.18-xen.img
           module /Q35_SINIT_17.BIN

```

GRUB2 does not pass the file name in the command line field of the multiboot entry (module_t::string). Since the tboot code is expecting the file name as the first part of the string, it tries to remove it to determine the command line arguments, which will cause a verification error. The "official" workaround for kernels/etc. that depend on the file name is to duplicate the file name in the grub.config file like below:

```c

	menuentry 'Xen w/ Intel(R) Trusted Execution Technology' {
	   recordfail
	   insmod part_msdos
	   insmod ext2
	   set root='(/dev/sda,msdos5)'
	   search --no-floppy --fs-uuid --set=root 4efb64c6-7e11-482e-8bab-07034a52de39
	   multiboot /tboot.gz /tboot.gz logging=vga,memory,serial
	   module /xen.gz /xen.gz iommu=required dom0_mem=524288 com1=115200,8n1
	   module /vmlinuz-2.6.18-xen /vmlinuz-2.6.18-xen root=/dev/VolGroup...
	   module /initrd-2.6.18-xen.img /initrd-2.6.18-xen.img
	   module /Q35_SINIT_17.BIN
	}

```

# TBoot in ASM Boot Stage

The code in `tboot-1.8.3/tboot/common/boot.S` does the following initalization:

Step 1. Setup the GDTR using the current CS reigster as the only guaranteed correct segement, all data segement is using the same segement.

Step 2. Set up the `bsp_stack` as the stack pointer.

Step 3. Reset EFLAGS (subsumes CLI and CLD).

Step 4. Preserve EAX to be a param to `begin_launch()` which contains either `MULTIBOOT_MAGIC` or `MULTIBOOT2_MAGIC`.

Step 5. Initialize BSS to all zero.

Step 6. Load IDTR and enable MCE.

Step 7. Pass multiboot info struct (in `EBX`), magic (in `EDX`) and call measured launch code `begin_launch()` which is the 1st C code in TBoot to be executed.

```c

	ENTRY(_stext)
	        jmp __start
			
			/*
             * NOTE1: There is a block of code that is ENTRY(_post_launch_entry) 
			 * which will be discussed later.
		     **/
	
	ENTRY(__start)
	        /* Set up a few descriptors: on entry only CS is guaranteed good. */
	        lgdt    %cs:gdt_descr
	        mov     $(ds_sel),%ecx
	        mov     %ecx,%ds
	        mov     %ecx,%es
	        mov     %ecx,%fs
	        mov     %ecx,%gs
	        mov     %ecx,%ss

	        ljmp    $(cs_sel),$(1f)
	1:	    leal	bsp_stack,%esp
	
	        /* Reset EFLAGS (subsumes CLI and CLD). */
	        pushl   $0
	        popf
	
	        /* preserve EAX to be a param to begin_launch--it should
	         * contain either MULTIBOOT_MAGIC or MULTIBOOT2_MAGIC--we'll need
	         * to figure out which */
	        mov     %eax,%edx
	
	        /* Initialize BSS (no nasty surprises!) */
	        mov     $__bss_start,%edi
	        mov     $_end,%ecx
	        sub     %edi,%ecx
	        xor     %eax,%eax
	        rep     stosb
	
	        /* Load IDT */
	        lidt    idt_descr
	
	        /* enable MCE */
	        mov     %cr4,%eax
	        or      $CR4_MCE,%eax
	        mov     %eax,%cr4
	
	        /* pass multiboot info struct, magic and call measured launch code */
	        push    %edx
	        push    %ebx
	        call    begin_launch
	        ud2

```

# TBoot Measured Launch Stage

The `begin_launch()` function is the first C function that starts the `Measured Launch Stage`. This section tries to describe the several initalization steps of the launch stage. This function is defined in `tboot-1.8.3/tboot/common/tboot.c`.

```c

	void begin_launch(void *addr, uint32_t magic)
	{
	    tb_error_t err;
	
	    if (g_ldr_ctx->type == 0)
	        determine_loader_type(addr, magic);
	
	    /* on pre-SENTER boot, copy command line to buffer in tboot image
	       (so that it will be measured); buffer must be 0 -filled */
	    if ( !is_launched() && !s3_flag ) {
	
	        const char *cmdline_orig = get_cmdline(g_ldr_ctx);
	        const char *cmdline = NULL;
	        if (cmdline_orig){
	            cmdline = skip_filename(cmdline_orig);
	        }
	        memset(g_cmdline, '\0', sizeof(g_cmdline));
	        if (cmdline)
	            strncpy(g_cmdline, cmdline, sizeof(g_cmdline)-1);
	    }
	
	    /* always parse cmdline */
	    tboot_parse_cmdline();
	
	    /* initialize all logging targets */
	    printk_init();
	
	    printk(TBOOT_INFO"******************* TBOOT *******************\n");
	    printk(TBOOT_INFO"   %s\n", TBOOT_CHANGESET);
	    printk(TBOOT_INFO"*********************************************\n");
	
	    printk(TBOOT_INFO"command line: %s\n", g_cmdline);
	    /* if telled to check revocation acm result, go with simplified path */
	    if ( get_tboot_call_racm_check() )
	        check_racm_result(); /* never return */
	
	    if ( s3_flag )
	        printk(TBOOT_INFO"resume from S3\n");
	
	    /* RLM scaffolding
	       if (g_ldr_ctx->type == 2)
	       print_loader_ctx(g_ldr_ctx);
	    */
	
	    /* clear resume vector on S3 resume so any resets will not use it */
	    if ( !is_launched() && s3_flag )
	        set_s3_resume_vector(&_tboot_shared.acpi_sinfo, 0);
	
	    /* we should only be executing on the BSP */
	    if ( !(rdmsr(MSR_APICBASE) & APICBASE_BSP) ) {
	        printk(TBOOT_INFO"entry processor is not BSP\n");
	        apply_policy(TB_ERR_FATAL);
	    }
	    printk(TBOOT_INFO"BSP is cpu %u\n", get_apicid());
	
	    /* make copy of e820 map that we will use and adjust */
	    if ( !s3_flag ) {
	        if ( !copy_e820_map(g_ldr_ctx) )
	            apply_policy(TB_ERR_FATAL);
	    }
	
	    /* we need to make sure this is a (TXT-) capable platform before using */
	    /* any of the features, incl. those required to check if the environment */
	    /* has already been launched */
	
	    /* make TPM ready for measured launch */
	    if ( !tpm_detect() )
	        apply_policy(TB_ERR_TPM_NOT_READY);
	
	    /* read tboot policy from TPM-NV (will use default if none in TPM-NV) */
	    err = set_policy();
	    apply_policy(err);
	
	    /* if telled to call revocation acm, go with simplified path */
	    if ( get_tboot_call_racm() )
	        launch_racm(); /* never return */
	
	    /* need to verify that platform supports TXT before we can check error */
	    /* (this includes TPM support) */
	    err = supports_txt();
	    apply_policy(err);
	
	    /* print any errors on last boot, which must be from TXT launch */
	    txt_get_error();
	
	    /* need to verify that platform can perform measured launch */
	    err = verify_platform();
	    apply_policy(err);
	
	    /* ensure there are modules */
	    if ( !s3_flag && !verify_loader_context(g_ldr_ctx) )
	        apply_policy(TB_ERR_FATAL);
	
	    /* this is being called post-measured launch */
	    if ( is_launched() )
	        post_launch();
	
	    /* make the CPU ready for measured launch */
	    if ( !prepare_cpu() )
	        apply_policy(TB_ERR_FATAL);
	
	    /* do s3 launch directly, if is a s3 resume */
	    if ( s3_flag ) {
	        if ( !prepare_tpm() )
	            apply_policy(TB_ERR_TPM_NOT_READY);
	        txt_s3_launch_environment();
	        printk(TBOOT_ERR"we should never get here\n");
	        apply_policy(TB_ERR_FATAL);
	    }
	
	    /* check for error from previous boot */
	    printk(TBOOT_INFO"checking previous errors on the last boot.\n\t");
	    if ( was_last_boot_error() )
	        printk(TBOOT_INFO"last boot has error.\n");
	    else
	        printk(TBOOT_INFO"last boot has no error.\n");
	
	    if ( !prepare_tpm() )
	        apply_policy(TB_ERR_TPM_NOT_READY);
	
	    /* launch the measured environment */
	    err = txt_launch_environment(g_ldr_ctx);
	    apply_policy(err);
	}

```

During the `begin_launch()` boot stage, TBoot will do the following:

Step 1. Save the boot loader `multiboot information` into `g_ldr_ctx`, with the `type` set to `MB1_ONLY` or `MB2_ONLY` accordingly (TBoot does not support other types and will be `doomed` for other types).

Step 2. On pre-SENTER boot, copy command line from `multiboot information` to `g_cmdline[]` buffer.

Step 3. The command line is parsed and saved into `g_tboot_param_values[]`, with default values as specified in `g_tboot_cmdline_options[]`.

```c

	/*
	 * the option names and default values must be separate from the actual
	 * params entered
	 * this allows the names and default values to be part of the MLE measurement
	 * param_values[] need to be in .bss section so that will get cleared on launch
	 */
	
	/* global option array for command line */
	static const cmdline_option_t g_tboot_cmdline_options[] = {
	    { "loglvl",     "all" },         /* all|err,warn,info|none */
	    { "logging",    "serial,vga" },  /* vga,serial,memory|none */
	    { "serial",     "115200,8n1,0x3f8" },
	    /* serial=<baud>[/<clock_hz>][,<DPS>[,<io-base>[,<irq>[,<serial-bdf>[,<bridge-bdf>]]]]] */
	    { "vga_delay",  "0" },           /* # secs */
	    { "ap_wake_mwait", "false" },    /* true|false */
	    { "pcr_map", "legacy" },         /* legacy|da */
	    { "min_ram", "0" },              /* size in bytes | 0 for no min */
	    { "call_racm", "false" },        /* true|false|check */
	    { "measure_nv", "false" },       /* true|false */
	    { "extpol",    "sha1" },        /* agile|embedded|sha1|sha256|sm3|... */
	    { NULL, NULL }
	};

	static char g_tboot_param_values[ARRAY_SIZE(g_tboot_cmdline_options)][MAX_VALUE_LEN];

```

Step 4. If the command line specifies `call_racm` to be `check`, the `begin_launch()` function tries to call `check_racm_result()` in turn it calls `txt_get_racm_error()` to dump the RACM (Revocation (RACM) SINIT) errors.

```c

	void check_racm_result(void)
	{
	    txt_get_racm_error();
	    shutdown_system(TB_SHUTDOWN_HALT); 
	}

	void txt_get_racm_error(void)
	{
	    txt_errorcode_t err;
	    acmod_error_t acmod_err;
	
	    /*
	     * display TXT.ERRORODE error
	     */
	    err = (txt_errorcode_t)read_pub_config_reg(TXTCR_ERRORCODE);
	
	    /* AC module error (don't know how to parse other errors) */
	    if ( err.valid == 0 ) {
	        printk(TBOOT_ERR
	               "Cannot retrieve status - ERRORSTS register is not valid.\n");
	        return;
	    } 
	
	    if ( err.external == 0 ) {      /* processor error */
	        printk(TBOOT_ERR"CPU generated error 0x%x\n", (uint32_t)err.type);
	        return;
	    }
	
	    acmod_err._raw = err.type;
	    if ( acmod_err.src == 1 ) {
	        printk(TBOOT_ERR"Unknown SW error.\n");
	        return;
	    }
	
	    if ( acmod_err.acm_type != 0x9 ) {
	        printk(TBOOT_ERR
	               "Cannot retrieve status - wrong ACM type in ERRORSTS register.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_ACM_ENTRY &&
	         acmod_err.error == ERR_TPM_DOUBLE_AUX ) {
	        printk(TBOOT_ERR
	               "Nothing to do: double AUX index is not valid TXT configuration.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_TPM_ACCESS &&
	         acmod_err.error == ERR_TPM_NV_INDEX_INVALID ) {
	        printk(TBOOT_ERR
	               "Nothing to do: invalid AUX index attributes.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_TPM_ACCESS &&
	         acmod_err.error == ERR_TPM_NV_PO_INDEX_INVALID ) {
	        printk(TBOOT_ERR
	               "Error: invalid PO index attributes.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_MISC_CONFIG &&
	         acmod_err.error == ERR_ALREADY_REVOKED ) {
	        printk(TBOOT_ERR
	               "Nothing to do: already revoked.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_MISC_CONFIG &&
	         acmod_err.error == ERR_FORBIDDEN_BY_OWNER ) {
	        printk(TBOOT_ERR
	               "Error: revocation forbidden by owner.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_MISC_CONFIG &&
	         acmod_err.error == ERR_CANNOT_REVERSE ) {
	        printk(TBOOT_ERR
	               "Error: cannot decrement revocation version.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_MISC_CONFIG &&
	         acmod_err.error == ERR_INVALID_RETURN_ADDR ) {
	        printk(TBOOT_ERR
	               "Error: invalid input address of return point.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == CLASS_MISC_CONFIG &&
	         acmod_err.error == ERR_NO_TPM ) {
	        printk(TBOOT_ERR
	               "Nothing to do: No TPM present.\n");
	        return;
	    }
	
	    if ( acmod_err.progress == 0 && acmod_err.error == 0 ) {
	        printk(TBOOT_INFO
	               "Success: Revocation completed.\n");
	        return;
	    }
	
	    printk(TBOOT_ERR"RACM generated error 0x%Lx.\n", err._raw);
	}

```

Step 5. The `begin_launch()` function calls `supports_txt()` to make sure the machine is capable of TXT operation. It verifies the processor is Intel CPU, and it should suppoprt both SMX and VMX operations; it then enables SMX operations to detect all the requires SMX features are available (otherwise it will disable SMX). The return result of `supports_txt()` identifies the type of error, such as `TB_ERR_SMX_NOT_SUPPORTED`, `TB_ERR_VMX_NOT_SUPPORTED`, `TB_ERR_TXT_NOT_SUPPORTED`, or `TB_ERR_NONE` when it is capable of supporting Intel TXT. This function is implemented in `tboot-1.8.3/tboot/txt/verify.c`.

```c

	tb_error_t supports_txt(void)
	{
	    capabilities_t cap;
	
	    /* processor must support cpuid and must be Intel CPU */
	    if ( !read_processor_info() )
	        return TB_ERR_SMX_NOT_SUPPORTED;
	
	    /* processor must support SMX */
	    if ( !supports_smx() )
	        return TB_ERR_SMX_NOT_SUPPORTED;
	
	    if ( use_mwait() ) {
	        /* set MONITOR/MWAIT support (SENTER will clear, so always set) */
	        uint64_t misc;
	        misc = rdmsr(MSR_IA32_MISC_ENABLE);
	        misc |= MSR_IA32_MISC_ENABLE_MONITOR_FSM;
	        wrmsr(MSR_IA32_MISC_ENABLE, misc);
	    }
	    else if ( !supports_vmx() ) {
	        return TB_ERR_VMX_NOT_SUPPORTED;
	    }
	
	    /* testing for chipset support requires enabling SMX on the processor */
	    write_cr4(read_cr4() | CR4_SMXE);
	    printk(TBOOT_INFO"SMX is enabled\n");
	
	    /*
	     * verify that an TXT-capable chipset is present and
	     * check that all needed SMX capabilities are supported
	     */
	
	    cap = __getsec_capabilities(0);
	    if ( cap.chipset_present ) {
	        if ( cap.senter && cap.sexit && cap.parameters && cap.smctrl &&
	             cap.wakeup ) {
	            printk(TBOOT_INFO"TXT chipset and all needed capabilities present\n");
	            return TB_ERR_NONE;
	        }
	        else
	            printk(TBOOT_ERR"ERR: insufficient SMX capabilities (%x)\n", cap._raw);
	    }
	    else
	        printk(TBOOT_ERR"ERR: TXT-capable chipset not present\n");
	
	    /* since we are failing, we should clear the SMX flag */
	    write_cr4(read_cr4() & ~CR4_SMXE);
	
	    return TB_ERR_TXT_NOT_SUPPORTED;
	}

```

Step 6. On returning from `supports_txt()` (or some other functions), the `begin_launch()` function calls `apply_policy()` to check the error and determine the next steps. It calls `write_tb_error_code()` to record the error condition into TPM NVRAM, and then `evaluate_error()` to check the potential action according to specific error types. Only when the action is `TB_POLACT_CONTINUE` can the code return and make the process continue. In the case of `TB_POLACT_UNMEASURED_LAUNCH`, it may still call `s3_launch()` (if `s3_flag` set), or call `launch_kernel(false)` to do a non-measured launch. The `apply_policy()` is defined in `tboot-1.8.3/tboot/common/policy.c`.

```c

	/*
	 * apply policy according to error happened.
	 */
	void apply_policy(tb_error_t error)
	{
	    tb_policy_action_t action;
	
	    /* save the error to TPM NV */
	    write_tb_error_code(error);
	
	    if ( error != TB_ERR_NONE )
	        print_tb_error_msg(error);
	
	    action = evaluate_error(error);
	    switch ( action ) {
	        case TB_POLACT_CONTINUE:
	            return;
	        case TB_POLACT_UNMEASURED_LAUNCH:
	            /* restore mtrr state saved before */
	            restore_mtrrs(NULL);
	            if ( s3_flag )
	                s3_launch();
	            else
	                launch_kernel(false);
	            break; /* if launch xen fails, do halt at the end */
	        case TB_POLACT_HALT:
	            break; /* do halt at the end */
	        default:
	            printk(TBOOT_ERR"Error: invalid policy action (%d)\n", action);
	            /* do halt at the end */
	    }
	
	    _tboot_shared.shutdown_type = TB_SHUTDOWN_HALT;
	    shutdown();
	}

```

Step 7. The `begin_launch()` function calls `txt_verify_platform()`, to check if `TXT_RESET.STS` is set, since if it is, the `SENTER` will fail.

```c
	
	tb_error_t txt_verify_platform(void)
	{
	    txt_heap_t *txt_heap;
	    tb_error_t err;
	
	    /* check TXT supported */
	    err = supports_txt();
	    if ( err != TB_ERR_NONE )
	        return err;
	
	    /* check is TXT_RESET.STS is set, since if it is SENTER will fail */
	    txt_ests_t ests = (txt_ests_t)read_pub_config_reg(TXTCR_ESTS);
	    if ( ests.txt_reset_sts ) {
	        printk(TBOOT_ERR"TXT_RESET.STS is set and SENTER is disabled (0x%02Lx)\n",
	               ests._raw);
	        return TB_ERR_SMX_NOT_SUPPORTED;
	    }
	
	    /* verify BIOS to OS data */
	    txt_heap = get_txt_heap();
	    if ( !verify_bios_data(txt_heap) )
	        return TB_ERR_TXT_NOT_SUPPORTED;
	
	    return TB_ERR_NONE;
	}

```

Step 8. The `begin_launch()` function calls `verify_loader_context()` to make sure the `multiboot information` contains `modules`, which are the `OS Kernel and Other Facilities` to be used. This is implemented in `tboot-1.8.3/tboot/common/loader.c`.

```c

	bool verify_loader_context(loader_ctx *lctx)
	{
	    unsigned int count;
	    if (LOADER_CTX_BAD(lctx))
	        return false;
	    count = get_module_count(lctx);
	    if (count < 1){
	        printk(TBOOT_ERR"Error: no MB%d modules\n", lctx->type);
	        return false;
	    } else
	        return true;
	}

	unsigned int 
	get_module_count(loader_ctx *lctx)
	{
	    if (LOADER_CTX_BAD(lctx))
	        return 0;
	    if (lctx->type == MB1_ONLY){
	        return(((multiboot_info_t *) lctx->addr)->mods_count);
	    } else {
	        /* currently must be type 2 */
	        struct mb2_tag *start = (struct mb2_tag *)(lctx->addr + 8);
	        unsigned int count = 0;
	        start = find_mb2_tag_type(start, MB2_TAG_TYPE_MODULE);
	        while (start != NULL){
	            count++;
	            /* nudge off this guy */
	            start = next_mb2_tag(start);
	            start = find_mb2_tag_type(start, MB2_TAG_TYPE_MODULE);
	        }
	        return count;
	    }
	}

```

Step 9. The `begin_launch()` function calls `txt_is_launched()` to check if the platform has already been launched. This checks the `SENTER.DONE.STS` bit in `TXT.STS – Status` register. The chipset sets this bit when it sees all of the threads have done an `TXT.CYC.SENTER-ACK`. When any of the threads does the `TXT.CYC.SEXIT-ACK` the `TXT.THREADS.JOIN` and `TXT.THREADS.EXISTS` registers will not be equal, so the chipset will clear this bit. If it is already launched, then it will call `post_launch()`, which will be discussed later.

```c

	bool txt_is_launched(void)
	{
	    txt_sts_t sts;
	
	    sts._raw = read_pub_config_reg(TXTCR_STS);
	
	    return sts.senter_done_sts;
	}

```

Step 10. If not already launched, the `begin_launch()` function calls `prepare_cpu()`, in turn calling `txt_prepare_cpu()` to prepare the CPU for launch. This is implemented in `tboot-1.8.3/tboot/txt/txt.c`. 

```c
	
	bool txt_prepare_cpu(void)
	{
	    unsigned long eflags, cr0;
	    uint64_t mcg_cap, mcg_stat;
	
	    /* must be running at CPL 0 => this is implicit in even getting this far */
	    /* since our bootstrap code loads a GDT, etc. */
	
	    cr0 = read_cr0();
	
	    /* must be in protected mode */
	    if ( !(cr0 & CR0_PE) ) {
	        printk(TBOOT_ERR"ERR: not in protected mode\n");
	        return false;
	    }
	
	    /* cache must be enabled (CR0.CD = CR0.NW = 0) */
	    if ( cr0 & CR0_CD ) {
	        printk(TBOOT_INFO"CR0.CD set\n");
	        cr0 &= ~CR0_CD;
	    }
	    if ( cr0 & CR0_NW ) {
	        printk(TBOOT_INFO"CR0.NW set\n");
	        cr0 &= ~CR0_NW;
	    }
	
	    /* native FPU error reporting must be enabled for proper */
	    /* interaction behavior */
	    if ( !(cr0 & CR0_NE) ) {
	        printk(TBOOT_INFO"CR0.NE not set\n");
	        cr0 |= CR0_NE;
	    }
	
	    write_cr0(cr0);
	
	    /* cannot be in virtual-8086 mode (EFLAGS.VM=1) */
	    eflags = read_eflags();
	    if ( eflags & X86_EFLAGS_VM ) {
	        printk(TBOOT_INFO"EFLAGS.VM set\n");
	        write_eflags(eflags | ~X86_EFLAGS_VM);
	    }
	
	    printk(TBOOT_INFO"CR0 and EFLAGS OK\n");
	
	    /*
	     * verify that we're not already in a protected environment
	     */
	    if ( txt_is_launched() ) {
	        printk(TBOOT_ERR"already in protected environment\n");
	        return false;
	    }
	
	    /*
	     * verify all machine check status registers are clear (unless
	     * support preserving them)
	     */
	
	    /* no machine check in progress (IA32_MCG_STATUS.MCIP=1) */
	    mcg_stat = rdmsr(MSR_MCG_STATUS);
	    if ( mcg_stat & 0x04 ) {
	        printk(TBOOT_ERR"machine check in progress\n");
	        return false;
	    }
	
	    getsec_parameters_t params;
	    if ( !get_parameters(&params) ) {
	        printk(TBOOT_ERR"get_parameters() failed\n");
	        return false;
	    }
	
	    /* check if all machine check regs are clear */
	    mcg_cap = rdmsr(MSR_MCG_CAP);
	    for ( unsigned int i = 0; i < (mcg_cap & 0xff); i++ ) {
	        mcg_stat = rdmsr(MSR_MC0_STATUS + 4*i);
	        if ( mcg_stat & (1ULL << 63) ) {
	            printk(TBOOT_ERR"MCG[%u] = %Lx ERROR\n", i, mcg_stat);
	            if ( !params.preserve_mce )
	                return false;
	        }
	    }
	
	    if ( params.preserve_mce )
	        printk(TBOOT_INFO"supports preserving machine check errors\n");
	    else
	        printk(TBOOT_INFO"no machine check errors\n");
	
	    if ( params.proc_based_scrtm )
	        printk(TBOOT_INFO"CPU support processor-based S-CRTM\n");
	
	    /* all is well with the processor state */
	    printk(TBOOT_INFO"CPU is ready for SENTER\n");
	
	    return true;
	}

```

Step 11. If `s3_flag` is set, the `begin_launch()` function will call `prepare_tpm()` [implemented in `tboot-1.8.3/tboot/common/tpm.c`], then call to `txt_s3_launch_environment()` [implemented in `tboot-1.8.3/tboot/txt/txt.c`] to do S3 launch. 

```c

	bool prepare_tpm(void)
	{
	    /*
	     * must ensure TPM_ACCESS_0.activeLocality bit is clear
	     * (: locality is not active)
	     */
	
	    return release_locality(0);
	}

	bool release_locality(uint32_t locality)
	{
	    uint32_t i;
	#ifdef TPM_TRACE
	    printk(TBOOT_DETA"TPM: releasing locality %u\n", locality);
	#endif
	
	    if ( !tpm_validate_locality(locality) )
	        return true;
	
	    tpm_reg_access_t reg_acc;
	    read_tpm_reg(locality, TPM_REG_ACCESS, &reg_acc);
	    if ( reg_acc.active_locality == 0 )
	        return true;
	
	    /* make inactive by writing a 1 */
	    reg_acc._raw[0] = 0;
	    reg_acc.active_locality = 1;
	    write_tpm_reg(locality, TPM_REG_ACCESS, &reg_acc);
	
	    i = 0;
	    do {
	        read_tpm_reg(locality, TPM_REG_ACCESS, &reg_acc);
	        if ( reg_acc.active_locality == 0 )
	            return true;
	        else
	            cpu_relax();
	        i++;
	    } while ( i <= TPM_ACTIVE_LOCALITY_TIME_OUT );
	
	    printk(TBOOT_INFO"TPM: access reg release locality timeout\n");
	    return false;
	}

	bool txt_s3_launch_environment(void)
	{
	    /* initial launch's TXT heap data is still in place and assumed valid */
	    /* so don't re-create; this is OK because it was untrusted initially */
	    /* and would be untrusted now */
	
	    /* initialize event log in os_sinit_data, so that events will not */
	    /* repeat when s3 */
	    if ( g_tpm->major == TPM12_VER_MAJOR && g_elog )
	        g_elog = (event_log_container_t *)init_event_log();
	    else if ( g_tpm->major == TPM20_VER_MAJOR && g_elog_2 )
	        init_evtlog_desc(g_elog_2);
	
	    /* get sinit binary loaded */
	    g_sinit = (acm_hdr_t *)(uint32_t)read_pub_config_reg(TXTCR_SINIT_BASE);
	    if ( g_sinit == NULL )
	        return false;
	
	    /* set MTRRs properly for AC module (SINIT) */
	    set_mtrrs_for_acmod(g_sinit);
	
	    printk(TBOOT_INFO"executing GETSEC[SENTER]...\n");
	    /* (optionally) pause before executing GETSEC[SENTER] */
	    if ( g_vga_delay > 0 )
	        delay(g_vga_delay * 1000);
	    __getsec_senter((uint32_t)g_sinit, (g_sinit->size)*4);
	    printk(TBOOT_ERR"ERROR--we should not get here!\n");
	    return false;
	}

```

Step 12. The normal boot launch in the `begin_launch()` function is done finally in `txt_launch_environment()`, as implemented in `tboot-1.8.3/tboot/txt/txt.c`. 

```c
	
	tb_error_t txt_launch_environment(loader_ctx *lctx)
	{
	    void *mle_ptab_base;
	    os_mle_data_t *os_mle_data;
	    txt_heap_t *txt_heap;
	
	    /*
	     * find correct SINIT AC module in modules list
	     */
	    find_platform_sinit_module(lctx, (void **)&g_sinit, NULL);
	    /* if it is newer than BIOS-provided version, then copy it to */
	    /* BIOS reserved region */
	    g_sinit = copy_sinit(g_sinit);
	    if ( g_sinit == NULL )
	        return TB_ERR_SINIT_NOT_PRESENT;
	    /* do some checks on it */
	    if ( !verify_acmod(g_sinit) )
	        return TB_ERR_ACMOD_VERIFY_FAILED;
	
	    /* print some debug info */
	    print_file_info();
	    print_mle_hdr(&g_mle_hdr);
	
	    /* create MLE page table */
	    mle_ptab_base = build_mle_pagetable(
	                             g_mle_hdr.mle_start_off + TBOOT_BASE_ADDR,
	                             g_mle_hdr.mle_end_off - g_mle_hdr.mle_start_off);
	    if ( mle_ptab_base == NULL )
	        return TB_ERR_FATAL;
	
	    /* initialize TXT heap */
	    txt_heap = init_txt_heap(mle_ptab_base, g_sinit, lctx);
	    if ( txt_heap == NULL )
	        return TB_ERR_TXT_NOT_SUPPORTED;
	
	    /* save MTRRs before we alter them for SINIT launch */
	    os_mle_data = get_os_mle_data_start(txt_heap);
	    save_mtrrs(&(os_mle_data->saved_mtrr_state));
	
	    /* set MTRRs properly for AC module (SINIT) */
	    if ( !set_mtrrs_for_acmod(g_sinit) )
	        return TB_ERR_FATAL;
	
	    printk(TBOOT_INFO"executing GETSEC[SENTER]...\n");
	    /* (optionally) pause before executing GETSEC[SENTER] */
	    if ( g_vga_delay > 0 )
	        delay(g_vga_delay * 1000);
	    __getsec_senter((uint32_t)g_sinit, (g_sinit->size)*4);
	    printk(TBOOT_INFO"ERROR--we should not get here!\n");
	    return TB_ERR_FATAL;
	}

```
