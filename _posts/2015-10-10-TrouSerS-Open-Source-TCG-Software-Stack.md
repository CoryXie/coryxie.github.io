---
layout: post
title: "TrouSerS - Open Source TCG Software Stack"
description:
headline:
modified: 2015-10-10
category: SecureDev
tags: [Security]
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

Demand for TPM support is increasing with more and more vendors selling low-cost discreet parts. [TrouSerS](http://trousers.sourceforge.net/) is an TCG Software Stack (TSS) that implements APIs to support TPM devices version 1.2 (and 1.1). It is licensed under a BSD type license. This blog entry describes the core concepts of TrouSerS.

## Trusted Platform Module

Trusted Platform Module (TPM) is an international standard for a secure cryptoprocessor, which is a dedicated microprocessor designed to secure hardware by integrating cryptographic keys into devices. In practice a TPM can be used for various different security applications such as secure boot and key storage. TPM is naturally supported only on devices that have TPM hardware support. If the hardware has TPM support but it is not showing up, it might need to be enabled in the BIOS settings. The following is an image for the layered architecture of a TPM enabled Intel Platform.

<img src="{{ site.url }}/images/2015-10-10-1/TPM-Platform.png" alt="TPM/TXT/TBoot/TrouSerS/OAT">

## TrouSerS

The TCG Software Stack provides an Application Programming Interface (API) to operating systems and applications so that they can use the functionality provided by a TPM. It allows different applications and even different operating systems, running as virtual machines on top of a Virtual Machine Monitor (VMM), to operate with each other, as long as they comply to the TSS specification. The main purpose of a TSS is to multiplex access to the TPM, since a TPM has limited resources and can only communicate with one client. Moreover, not all functions related to Trusted Computing require TPM access. These functions are located within the TSS.

TrouSerS is an open source TSS implementation maintained by IBM since 2004. TrouSerS works in a client-server fashion where the server is the only part of the system having access to the TPM itself. The other part is a library that provides standard APIs for accessing the TPM via the server only. The server (TCSD) is made of the following parts:

- tcsd - User Space Daemon as only portal to TPM device driver
- tcs - TSS Core Service
- tddl - TCG Device Driver Library

All clients use a TrouSerS library containing the following:

- trspi - TrouSerS low level interfaces for TSPI
- tspi - TSS Service Provider Interface

The TSS is comprised of three layers, as demonstrated in the following figure.

<img src="{{ site.url }}/images/2015-10-10-1/TSS-Layers.png" alt="Example object model of a TSS instance">

A more detailed view of the TSS components is provided below.

<img src="{{ site.url }}/images/2015-10-10-1/TSS-Components.png" alt="TSS Components">

### TCSD

TPM is managed by `tcsd`, a userspace daemon that manages Trusted Computing resources and should be (according to the TSS spec) the only portal to the TPM device driver. `tcsd` can be configured via `/etc/tcsd.conf`.

### TSP and TSPI

Every application employing TPM functionality has to use a TSS Service Provider (TSP) via its interface, the TSS Service Provider Interface (TSPI). Every application uses its own TSP, which is loaded as a library into any application process making use of it. The TSP provides high-level TCG functions as well as auxiliary services
such as hashing or signature verification. The TSP is a must for every TSS implementation. The TSP can be a discrete module; it is possible to integrate the TSP into other platform modules. One aspect of the TSP is the TSP Context Manager (TSPCM), which provides dynamic handles allowing efficient usage of TSPs and application resources. A second aspect is the TSP Cryptographic Functions (TSPCF), which are provided to make full use of the protected functions in a TPM. The TSPI is defined as a C interface.

### TCS and TCSI

The TSS Core Service (TCS) provides a common set of operations to all of the TSPs running on a platform. Since there is only one instance of a TCS per TPM, the main task of the TCS is to multiplex TPM access. Moreover, the TCS provides operations to store, manage and protect keys as well as privacy-sensitive credentials associated with the platform. The TCS is implemented as a user space daemon.

### TDDL and TDDLI

The TCG Device Driver Library (TDDL), accessed via the TCG Device Driver Library Interface (TDDLI), provides a unique interface to different TPM driver implementations. Moreover, it provides a transition between kernel mode and user mode. The TDDL provides functions (e.g. Open, Close, GetStatus) to maintain communication with the TDD. Additionally, it provides functions (e.g. GetCapability, SetCapability) to get and set attributes of the TPM, TDD and TDDL as well as direct functions (e.g. Transmit, Cancel) to transmit and cancel TPM commands.

### TDD and TDDI

The TPM Device Driver incorporates code that understands the specific behavior of the TPM. It is the only module that communicates directly with the TPM. It receives byte streams from the TDDL and sends them on to the TPM, returning its response back to the TDDL. Although the TSS specification mentions the TDD interface
TDDI, the current TSS specification does not define it. Different from the TDDL, which is implemented in user space, the TDD is implemented in kernel space.

## TSS and Linux TDD

The interfaces involved in the communication between TSS and Linux TDD are described in this section.

In order to exchange data between the TrouSerS and the TPM, TrouSerS makes use of its built-in TDDL, which is responsible for the communication with the Linux TPM device drivers. The main functions offered by the TDDL are
specified in the TSS 1.2 API.

Since the communication between the TSS and the Linux TDD is handled within the TDDL, all necessary functions are implemented in a library called `libtddl.a`, which is only used by the TCS daemon. The source code of this library is located inside the TrouSerS code base under `trousers-0.3.13/src/tddl/tddl.c`.

Upon starting TrouSerS, a TCS daemon is loaded that first tries to identify the availability of a TPM in the system. This is done by looking for certain device nodes on the device file system. In detail, the `Tddli_Open()` function looks for `/dev/tpm`, `/dev/tpm0` and `/udev/tpm0`. If found, the device is opened and thereby blocked for any other application trying to use this specific TPM.

In case an application uses the TSS, a byte buffer is generated that contains the TPM function call including all necessary parameters to carry out the request. The `tcsd` then communicates with the TPM device driver using the `Tddli_TransmitData()` function. This sends the data in the buffer to the corresponding TPM device driver
device node on the device file system and waits for the TPM device driver to complete the request. After the TPM device driver has passed the data to the TPM, it places the response from the TPM on the same device node, which the `Tddli_TransmitData()` function reads and returns the response. During termination of the `tcsd`, the function
`Tddli_Close()` unlocks the TPM device driver. 

The following figure describes the communication sequence between TrouSerS and the TPM.

<img src="{{ site.url }}/images/2015-10-10-1/TDDI-TPM.png" alt="Communication between TrouSerS TDDI and TPM">


### TPM services provided by the TrouSerS TSS API

- RSA key pair generation
- RSA encryption and decryption using PKCS v1.5 and OAEP padding
- RSA sign/verify
- Extend data into the TPM's PCRs and log these events
- Seal data to arbitrary PCRs
- Random Number Generation
- RSA key storage

## Credits

Parts of this blog entry are edits of contents found in the [Introduction and Analysis of the Open Source TCG Software Stack TrouSerS and Tools in its Environment](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/Studies/TSS_Apps/TSS-Apps_en.pdf).












