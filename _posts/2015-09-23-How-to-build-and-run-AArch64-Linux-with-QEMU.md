---
layout: post
title: "How to build and run AArch64 Linux with QEMU?"
description: 
headline: 
modified: 2015-09-23
category: KernelDevel
tags: [Kernel]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

I am going to start a series of blog entries recording my experiences on trying out (and source code reading/studying) for the ARMv8 AArch64 Linux. Here is the 1st one that describes the steps to build latest kernel and rootfs (from the latest git tree), then boot it using QEMU. Debugging is not discussed in this blog entry and will be covered in a latter blog.

---

## 1. Build qemu.git

> sudo apt-get build-dep qemu
> git clone git://git.qemu.org/qemu.git qemu.git
> cd qemu.git
> ./configure 
> make -j
> sudo make install

## 2. Build buildroot.git

> git clone git://git.buildroot.net/buildroot buildroot.git
> cd buildroot.git
> make menuconfig

There are lots of configuration options to choose from but the following are what I use:

> * Target Options -> Target Architecture(AArch64)
> * Toolchain -> Toolchain type (External toolchain)
> * Toolchain -> Toolchain (Linaro AArch64 14.09)
> * System configuration -> Run a getty (login prompt) after boot (BR2_TARGET_GENERIC_GETTY)
> * System configuration -> getty options -> TTY Port (ttyAMA0) (BR2_TARGET_GENERIC_GETTY_PORT)
> * Target Packages -> Show packages that are also provided by busybox (BR2_PACKAGE_BUSYBOX_SHOW_OTHERS)
> * Filesystem images -> cpio the root filesystem (for use as an initial RAM filesystem) (BR2_TARGET_ROOTFS_CPIO)

The last one will be important when we build the kernel next. Once you have configured buildroot it's time to type make and leave it for a while as you enjoy a nice lunch.

## 3. Build linux.git

> git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git linux.git

> cd linux.git

> ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig

> ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make menuconfig 

> * CONFIG_CROSS_COMPILE="aarch64-linux-gnu-"   # needs to match your cross-compiler prefix
> * CONFIG_INITRAMFS_SOURCE="/home/coryxie/projects/arm64/buildroot.git/output/images/rootfs.cpio"  # points at your buildroot image
> * CONFIG_NET_9P=y     # needed for virtfs mount
> * CONFIG_NET_9P_VIRTIO=y

> ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j 8

## 4. Run kernel in QEMU

> qemu-system-aarch64 -machine virt -cpu cortex-a57 -machine type=virt -nographic -smp 1 -m 128 -kernel arch/arm64/boot/Image  --append "console=ttyAMA0" -fsdev local,id=r,path=/home/coryxie/projects/arm64,security_model=none -device virtio-9p-device,fsdev=r,mount_tag=r 

> * Note that we can add two options "-s -S" in the end for debugging with Eclipse which will be addressed in another post.

In the QEMU, kernel boots and goes to the prompt:

    coryxie@ubuntu:~/projects/arm64/linux.git$ qemu-system-aarch64 -s -S -machine virt -cpu cortex-a57 -machine type=virt -nographic -smp 1 -m 128 -kernel arch/arm64/boot/Image  --append "console=ttyAMA0" -fsdev local,id=r,path=/home/coryxie/projects/arm64,security_model=none -device virtio-9p-device,fsdev=r,mount_tag=r
    Booting Linux on physical CPU 0x0
    Initializing cgroup subsys cpu
    Linux version 4.0.0+ (coryxie@ubuntu) (gcc version 4.9.2 20140904 (prerelease) (crosstool-NG linaro-1.13.1-4.9-2014.09 - Linaro GCC 4.9-2014.09) ) #4 SMP PREEMPT Sat Apr 18 09:27:56 PDT 2015
    CPU: AArch64 Processor [411fd070] revision 0
    Detected PIPT I-cache on CPU0
    efi: Getting EFI parameters from FDT:
    efi: UEFI not found.
    cma: Reserved 16 MiB at 0x0000000047000000
    psci: probing for conduit method from DT.
    psci: PSCIv0.2 detected in firmware.
    psci: Using standard PSCI v0.2 function IDs
    PERCPU: Embedded 16 pages/cpu @ffffffc006fe3000 s27968 r8192 d29376 u65536
    Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 32256
    Kernel command line: console=ttyAMA0
    PID hash table entries: 512 (order: 0, 4096 bytes)
    Dentry cache hash table entries: 16384 (order: 5, 131072 bytes)
    Inode-cache hash table entries: 8192 (order: 4, 65536 bytes)
    Cannot allocate SWIOTLB buffer
    Memory: 88220K/131072K available (5069K kernel code, 333K rwdata, 1892K rodata, 2164K init, 201K bss, 26468K reserved, 16384K cma-reserved)
    Virtual kernel memory layout:
        vmalloc : 0xffffff8000000000 - 0xffffffbdbfff0000   (   246 GB)
        vmemmap : 0xffffffbdc0000000 - 0xffffffbfc0000000   (     8 GB maximum)
                  0xffffffbdc1000000 - 0xffffffbdc1200000   (     2 MB actual)
        fixed   : 0xffffffbffabfe000 - 0xffffffbffac00000   (     8 KB)
        PCI I/O : 0xffffffbffae00000 - 0xffffffbffbe00000   (    16 MB)
        modules : 0xffffffbffc000000 - 0xffffffc000000000   (    64 MB)
        memory  : 0xffffffc000000000 - 0xffffffc008000000   (   128 MB)
          .init : 0xffffffc00074f000 - 0xffffffc00096c000   (  2164 KB)
          .text : 0xffffffc000080000 - 0xffffffc00074e544   (  6970 KB)
          .data : 0xffffffc000970000 - 0xffffffc0009c3600   (   334 KB)
    SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
    Preemptible hierarchical RCU implementation.
    	Additional per-CPU info printed with stalls.
    	RCU restricting CPUs from NR_CPUS=64 to nr_cpu_ids=1.
    RCU: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=1
    NR_IRQS:64 nr_irqs:64 0
    Architected cp15 timer(s) running at 62.50MHz (virt).
    clocksource arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
    sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
    Console: colour dummy device 80x25
    Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=625000)
    pid_max: default: 32768 minimum: 301
    Security Framework initialized
    Mount-cache hash table entries: 512 (order: 0, 4096 bytes)
    Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes)
    Initializing cgroup subsys memory
    Initializing cgroup subsys hugetlb
    hw perfevents: no hardware support available
    EFI services will not be available.
    Brought up 1 CPUs
    SMP: Total of 1 processors activated.
    devtmpfs: initialized
    DMI not present or invalid.
    clocksource jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
    NET: Registered protocol family 16
    cpuidle: using governor ladder
    cpuidle: using governor menu
    vdso: 2 pages (1 code @ ffffffc000975000, 1 data @ ffffffc000974000)
    hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
    DMA: preallocated 256 KiB pool for atomic allocations
    Serial: AMBA PL011 UART driver
    9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 38, base_baud = 0) is a PL011 rev1
    console [ttyAMA0] enabled
    vgaarb: loaded
    SCSI subsystem initialized
    usbcore: registered new interface driver usbfs
    usbcore: registered new interface driver hub
    usbcore: registered new device driver usb
    Switched to clocksource arch_sys_counter
    NET: Registered protocol family 2
    TCP established hash table entries: 1024 (order: 1, 8192 bytes)
    TCP bind hash table entries: 1024 (order: 2, 16384 bytes)
    TCP: Hash tables configured (established 1024 bind 1024)
    TCP: reno registered
    UDP hash table entries: 256 (order: 1, 8192 bytes)
    UDP-Lite hash table entries: 256 (order: 1, 8192 bytes)
    NET: Registered protocol family 1
    RPC: Registered named UNIX socket transport module.
    RPC: Registered udp transport module.
    RPC: Registered tcp transport module.
    RPC: Registered tcp NFSv4.1 backchannel transport module.
    kvm [1]: HYP mode not available
    futex hash table entries: 256 (order: 2, 16384 bytes)
    audit: initializing netlink subsys (disabled)
    audit: type=2000 audit(0.490:1): initialized
    HugeTLB registered 2 MB page size, pre-allocated 0 pages
    VFS: Disk quotas dquot_6.5.2
    VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
    fuse init (API version 7.23)
    9p: Installing v9fs 9p2000 file system support
    io scheduler noop registered
    io scheduler cfq registered (default)
    Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
    Unable to detect cache hierarcy from DT for CPU 0
    loop: module loaded
    tun: Universal TUN/TAP device driver, 1.6
    tun: (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
    ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
    ehci-pci: EHCI PCI platform driver
    ehci-platform: EHCI generic platform driver
    ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
    ohci-pci: OHCI PCI platform driver
    ohci-platform: OHCI generic platform driver
    usbcore: registered new interface driver usb-storage
    mousedev: PS/2 mouse device common for all mice
    Driver 'mmcblk' needs updating - please use bus_type methods
    sdhci: Secure Digital Host Controller Interface driver
    sdhci: Copyright(c) Pierre Ossman
    sdhci-pltfm: SDHCI platform and OF driver helper
    usbcore: registered new interface driver usbhid
    usbhid: USB HID core driver
    TCP: cubic registered
    NET: Registered protocol family 17
    9pnet: Installing 9P2000 support
    registered taskstats version 1
    drivers/rtc/hctosys.c: unable to open rtc device (rtc0)
    Freeing unused kernel memory: 2164K (ffffffc00074f000 - ffffffc00096c000)
    Freeing alternatives memory: 8K (ffffffc00096c000 - ffffffc00096e000)
    Starting logging: OK
    Initializing random number generator... random: dd urandom read with 1 bits of entropy available
    done.
    Starting network...
    Welcome to Buildroot
    buildroot login: root
    # ls /mnt/
    # mount -t 9p -o trans=virtio r /mnt
    # ls /mnt/
    buildroot.git     linux-4.0.tar.xz  qemu.git
    linux             linux.git
    # mount -t debugfs none /sys/kernel/debug






