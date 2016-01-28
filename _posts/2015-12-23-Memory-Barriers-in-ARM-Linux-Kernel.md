---
layout: post
title: "Memory Barriers in the ARM Linux Kernel"
description: 
headline: 
modified: 2015-12-23
category: KernelDev
tags: [KernelDev]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

The ARMv7/v8 architectures feature weakly-ordered memory models, allowing hardware designers to implement a variety of optimisations in the memory subsystem. Whilst this can improve the performance and power consumption of embedded ARM CPUs, it can also lead to subtle software bugs which are incredibly hard to debug. As a result, software engineers tend to use either too many barrier instructions or heavyweight variants to err on the side of caution. Both of these practices have performance costs, which will increase as the number of cores on a typical SoC continues to rise. *W. Deacon* from ARM made a presentation which described the various barrier instructions in the ARM architecture and how to use them in the Linux kernel. It will also introduce changes proposed to the ARM kernel port allowing users of barriers to control their propagation within the system and measurably improve performance.

# Effective Use of Memory Barriers in the ARM Linux Kernel

* This is the Youtube Video of the talk.

<center> <iframe src="https://www.youtube.com/embed/6ORn6_35kKo" frameborder="0" allowfullscreen>From Weak to Weedy: Effective Use of Memory Barriers in the ARM Linux Kernel - W. Deacon, ARM</iframe><center>

<center> <iframe src="http://events.linuxfoundation.org/sites/events/files/slides/weak-to-weedy.pdf" width="800" height="600">From Weak to Weedy: Effective Use of Memory Barriers in the ARM Linux Kernel - W. Deacon, ARM</iframe> </center>
