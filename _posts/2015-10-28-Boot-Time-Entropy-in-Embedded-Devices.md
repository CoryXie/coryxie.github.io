---
layout: post
title: "Boot-Time Entropy in Embedded Devices"
description:
headline:
modified: 2015-10-28
category: PaperTips
tags: [Embedded]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

Random numbers unpredictable by an adversary are crucial to many computing tasks. But computers are designed to be deterministic, which makes it difficult to generate random numbers. The Linux kernel randomness subsystem exposes a blocking interface (`/dev/random`) and a nonblocking interface (`/dev/urandom`) and nearly all software uses the nonblocking interface. The timing of every IRQ is now an entropy source. Entropy is first applied to the nonblocking pool, in the hope of supplying randomness to clients soon after boot. (Clients waiting on the blocking interface can block a bit longer.) However, it is observed that there is still race condition between entropy accumulation and the reading of supposedly random bytes from the nonblocking pool. The paper <`Welcome to the Entropics: Boot-Time Entropy in Embedded Devices`> describes 3 ways to gather entropy so early in the boot process that all requests for randomness can be satisfied.

# EARLY KERNEL ENTROPY MECHANISMS


1) To instrument the kernel startup code to record how long each block of code takes to execute. This approach has previously been used to gather entropy in userland code; it is shown that it is also applicable when a single kernel thread of execution runs, with interrupts disabled, on an embedded system.

2) To take advantage of architectural features that vary between SoCs, rendering them less portable and less widely applicable, but promising more entropy.

3) It is possible for bootloader code, running from on-chip SRAM, to turn off DRAM refresh. With refresh disabled, the contents of DRAM decay unpredictably; we exploit this fact to obtain an entropy source.

# ARCHITECTURAL CAUSES OF TIMING VARIATION

Two physical mechanisms are testified that can partly explain the non-determinism the researchers measured during
the execution of early kernel code: communication latency (variation that can arise while sending data between two
clock domains) and memory latency (variation that arises due to interactions with DRAM refresh). Other mechanisms we do not understand are likely also involved.

1) Clock Domain Crossing

Processor designs use multiple clock domains to allow different portions of the chip to run at different frequency. Due to misalignment between the domains, the amount of time it takes to send messages between two clock domains can vary.

2) DRAM Decay

Because DRAM bits decay over time, the system must periodically read and re-write each DRAM storage location. Depending on the system, the processor memory controller issues refresh commands or, alternately, the processor can place the chips in an auto-refresh mode so they handle refresh autonomously. The most useful source of randomness the researchers found is the decay of data stored in DRAM over time. DRAM decay occurs when the charge that stores a binary value leaks off the capacitor in a DRAM storage cell. The decay rate of DRAM bits varies widely as a result of manufacturing variation, temperature, the data stored in the cell, and other factors.

3) PLL Lock Latency

The PLLs that produce the on-chip clocks in modern processors are complex, analog devices. When they start up (or the chip reconfigures them), they take a variable amount of time to "lock" on to the new output frequency. This variation in lock time is due to a number of factors, including stability of the power supply, accuracy and jitter in the source oscillator, temperature, and manufacturing process variation. Repeatedly reconfiguring an on-chip PLL and measuring how long it takes to lock will result in random variations.

# CONCLUSION


The first technique, which times the execution of kernel code blocks, provides a moderate amount of entropy and is
easily applied to every system we examined, but we are able to give only a partial account for the source of the entropy it gathers.

The second technique, DRAM decay, provides a large amount of entropy, but presents a heavy performance penalty and is tricky to deploy, relying on details of the memory controller. Its advantage is a physical justification for the
observed randomness.

The third technique, timing PLL locking, promises the highest bitrate and is well supported by physical processes, but its implementation requires intimate knowledge of the individual SoC.

# References

This blog is a note edit from the paper [http://cseweb.ucsd.edu/~swanson/papers/Oakland2013EarlyEntropy.pdf](http://cseweb.ucsd.edu/~swanson/papers/Oakland2013EarlyEntropy.pdf "Welcome to the Entropics: Boot-Time Entropy in Embedded Devices"). 

