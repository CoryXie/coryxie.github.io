---
layout: post
title: "Linux Kernel CONFIG_LIVEPATCH - Kernel Live Patching"
description:
headline:
modified: 2015-10-27
category: KernelDev
tags: [Kernel]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

Since Linux Kernel 4.0 release, the live kernel patching capability is now a mainstream feature in the Linux ecosystem. The live kernel patching capability integrated into the Linux 4.0 kernel is the result of a joint effort between Red Hat (`kpatch`) and SUSE (`kgraft`) to bring their respective approaches together. It has obsoleted (in some degree) the similar Oracle technology known as `Ksplice` that also enables live kernel patching. This blog entry is to describe how `CONFIG_LIVEPATCH` works from the source code level.

# Config Option

The live kernel patching is enabled by the `CONFIG_LIVEPATCH` configuration , as can be seen from `linux-4.2/kernel/livepatch/Kconfig`.

```c

	config HAVE_LIVEPATCH
		bool
		help
		  Arch supports kernel live patching
	
	config LIVEPATCH
		bool "Kernel Live Patching"
		depends on DYNAMIC_FTRACE_WITH_REGS
		depends on MODULES
		depends on SYSFS
		depends on KALLSYMS_ALL
		depends on HAVE_LIVEPATCH
		help
		  Say Y here if you want to support kernel live patching.
		  This option has no runtime impact until a kernel "patch"
		  module uses the interface provided by this option to register
		  a patch, causing calls to patched functions to be redirected
		  to new function code contained in the patch module.

```

* **`HAVE_LIVEPATCH`** 
 
Besides requiring the core `CONFIG_LIVEPATCH`, each CPU architecture is required to have corresponding support code, as indicated by the each supported architecture to select `HAVE_LIVEPATCH` option. Right now in the Linux 4.2 release there are only two architectures selected this option, the x86 (only `X86_64`) and s390, by `Kconfig` with a line of `select HAVE_LIVEPATCH if X86_64`.

* **`DYNAMIC_FTRACE_WITH_REGS`**

The live kernel patching is an `ftrace`-based mechanism and kernel interface for doing live patching of kernel and kernel module functions. The `ftrace` (abbreviated from `Function Tracer`) is a tracing framework for the Linux kernel. The original ability of `ftrace` is to record information related to various function calls performed while the kernel is running, but it can do more.



 