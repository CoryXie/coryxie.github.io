---
layout: post
title: "Xvisor ARMv8 Startup Analysis"
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

Xvisor is an open-source type-1 hypervisor, which aims at providing a monolithic, light-weight, portable, and flexible virtualization solution. It provides a high performance and low memory foot print virtualization solution for ARMv5, ARMv6, ARMv7a, ARMv7a-ve, ARMv8a, x86_64, and other CPU architectures. In comparison to other ARM hypervisors, it is the only hypervisor providing support for ARM CPUs which do not have ARM virtualization extensions. Some time ago when I was first reading the ARMv8 port of Xvisor I made some analysis of the ARMv8 startup process (some Chinese included but should be mostly readable since most of them are captuering some images from ARM TRM reading). The following is the PDF version!

# Xvisor ARMv8 `foundation_v8_boot.S` Analysis

<center> <iframe src="{{ site.baseurl }}/images/2015-12-23-1/Xvisor-ARMv8-Startup.pdf" width="1600" height="800">XVisor ARMv8 Startup Analysis</iframe> </center>
