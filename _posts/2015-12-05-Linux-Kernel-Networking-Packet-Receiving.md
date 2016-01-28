---
layout: post
title: "Linux Kernel Networking Packet Receiving Process"
description: 
headline: 
modified: 2015-12-05
category: NetworkingDev
tags: [Networing]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

Recent trends in technology are showing that although the raw transmission speeds used in networks are increasing rapidly, the rate of advancement of microprocessor technology has slowed down. Therefore, network protocol processing overheads have risen sharply in comparison with the time spent in packet transmission, resulting in degraded throughput for networked applications. More and more, it is the network end system, instead of the network, that is responsible for degraded performance of network applications. In this blog post, the Linux system’s packet receive process is studied from NIC to application.

## The Birth of Kernel Networking "New API"

The network technology at layers 1 and 2 assumes an Ethernet medium, since it is the most widespread and representative LAN technology. Also, it is assumed that the Ethernet device driver makes use of Linux’s "New API" or NAPI, which reduces the interrupt load on the CPUs.

"New API" (also referred to as NAPI) is an interface to use interrupt mitigation techniques for networking devices in the Linux kernel. Such an approach is intended to reduce the overhead of packet receiving. The idea is to defer incoming message handling until there is a sufficient amount of them so that it is worth handling them all at once.

A straightforward method of implementing a network driver is to interrupt the kernel by issuing an interrupt request (IRQ) for each and every incoming packet. However, servicing IRQs is costly in terms of processor resources and time. Therefore the straightforward implementation can be very inefficient in high-speed networks, constantly interrupting the kernel with the thousands of packets per second. Overall performance of the system as well as network throughput can suffer as a result.

There are some subtle problems with using softirqs and tasklets. Some are obvious – driver writers must be more careful with locking. Other problems are less obvious. For example, it’s a great thing to be able to take interrupts on all CPUs simultaneously, but there’s no point in taking an interrupt if it can’t be processed before the next one is received.

Networking is particularly vulnerable to this. Assuming the interrupt controller distributes interrupts among CPUs in a round-robin fashion (this is the default for Intel IO-APICs), worst-case behaviour can be produced by simply ping-flooding an SMP machine. Interrupts will hit each CPU in turn, raising the network receive softirq. Each CPU will then attempt to deliver its packet into the networking stack. Even if the CPUs don’t spend all their time spinning on locks waiting for each other to exit critical regions, they steal cache lines from each other and waste time that way.

Advanced network cards implement a feature called interrupt mitigation. Instead of interrupting the CPU for each packet received, they queue packets in their on-card RAM and only generate an interrupt when a sufficient number of packets have arrived. The NAPI work, done by *Jamal Hadi Salim*, *Alexey Kuznetsov* and *Thomas Olsson*, simulates this in the OS.

When the network card driver receives a packet, it calls `disable_irq()` before passing the packet to the network stack’s receive softirq. After the network stack has processed the packet, it asks the driver whether any more packets have arrived in the meantime. If none have, the driver calls `enable_irq()`. Otherwise, the network stack processes the new packets and leaves the network card’s interrupt disabled. This effectively leaves the card in polling mode, and prevents any card from consuming too much of the system’s resources.

Polling is an alternative to interrupt-based processing. The kernel can periodically check for the arrival of incoming network packets without being interrupted, which eliminates the overhead of interrupt processing. Establishing an optimal polling frequency is important, however. Too frequent polling wastes CPU resources by repeatedly checking for incoming packets that have not yet arrived. On the other hand, polling too infrequently introduces latency by reducing system reactivity to incoming packets, and it may result in the loss of packets if the incoming packet buffer fills up before being processed.

As a compromise, the Linux kernel uses the interrupt-driven mode by default and only switches to polling mode when the flow of incoming packets exceeds a certain threshold, known as the "weight" of the network interface.

## Packet Receiving Process

The following figure demonstrates generally the trip of a packet from its ingress into a Linux end system to its final delivery to the application. 

<img src="{{ site.baseurl }}/images/2015-12-05-1/packet-receive-process.png" alt="Networking Packet Receive Process">

In general, the packet’s trip can be classified into three stages:

>- Packet is transferred from network interface card (NIC) to ring buffer. The NIC and device driver manage and control this process. 
>- Packet is transferred from ring buffer to a socket receive buffer, driven by a software interrupt request(softirq). The kernel protocol stack handles this stage.
>- Packet data is copied from the socket receive buffer to the application, which we will term the Data Receiving Process.

In the following sections, we detail these three stages. 

### NIC and Device Driver Processing

The NIC and its device driver perform the layer 1 and 2 functions of the OSI 7-layer network model: packets are received and transformed from raw physical signals, and placed into system memory, ready for higher layer processing. The Linux kernel uses a structure `sk_buff` to hold any single packet up to the `MTU (Maximum Transfer Unit)` of the network. The device driver maintains a "ring" of these packet buffers, known as a "ring buffer", for packet reception (and a separate ring for transmission). A ring buffer consists of a device and driver dependent number of packet descriptors. To be able to receive a packet, a packet descriptor should be in "ready" state, which means it has been initialized and pre-allocated with an empty `sk_buff` which has been memory-mapped into address
space accessible by the NIC over the system I/O bus. When a packet comes, one of the ready packet descriptors in the receive ring will be used, the packet will be transferred by DMA into the pre-allocated `sk_buff`, and the descriptor will be marked as used. A used packet descriptor should be reinitialized and refilled with an empty `sk_buff` as soon as possible for further incoming packets. If a packet arrives and there is no ready packet
descriptor in the receive ring, it will be discarded. Once a packet is transferred into the main memory, during subsequent processing in the network stack, the packet remains at the same kernel memory location. 

The following figure shows a general packet receiving process at NIC and device driver level. When a packet is received, it is transferred into main memory and an interrupt is raised only after the packet is accessible to
the kernel. When CPU responds to the interrupt, the driver’s interrupt handler is called, within which the 
`softirq` is scheduled. It puts a reference to the device into the `poll queue` of the interrupted CPU. The interrupt handler also disables the NIC’s receive interrupt till the packets in its ring buffer are processed.

<img src="{{ site.baseurl }}/images/2015-12-05-1/nic-device-driver-process.png" alt="Networking NIC & Device Driver Processing Steps">

The `softirq` is serviced shortly afterward. The CPU polls each device in its `poll queue` to get the received packets from the ring buffer by calling the `poll` method of the device driver. Each received packet is passed upwards for further protocol processing. After a received packet is dequeued from its receive ring buffer for further processing, its corresponding packet descriptor in the receive ring buffer needs to be reinitialized and refilled.

### Kernel Protocol Stack

#### IP Processing

The IP protocol receive function is called during the processing of a `softirq` for each IP packet that is dequeued from the ring buffer. This function performs initial checks on the IP packet, which mainly involve verifying its integrity, applying firewall rules, and disposing the packet for forwarding or local delivery to a higher level protocol. For each transport layer protocol, a corresponding handler function is defined: `tcp_v4_rcv()` and
`udp_rcv()` are two examples. 

#### TCP Processing

When a packet is handed upwards for TCP processing, the function `tcp_v4_rcv()` first performs the TCP header processing. Then the socket associated with the packet is determined, and the packet dropped if none exists. A socket has a `lock` structure to protect it from un-synchronized access. If the socket is locked, the packet waits on the `backlog queue` before being processed further. If the socket is not locked, and its `Data Receiving Process` is sleeping for data, the packet is added to the socket’s `prequeue` and will be processed in batch in
the process context, instead of the interrupt context. Placing the first packet in the `prequeue` will wake up the
sleeping data receiving process. If the `prequeue` mechanism does not accept the packet, which means that the socket is not locked and no process is waiting for input on it, the packet must be processed immediately by a call to `tcp_v4_do_rcv()`. The same function also is called to drain the `backlog queue` and `prequeue`. Those queues (except in the case of `prequeue overflow`) are drained in the process context, not the interrupt context of the `softirq`. In the case of `prequeue overflow`, which means that packets within the `prequeue` reach/exceed the socket’s receive buffer quota, those packets should be processed as soon as possible, even in the interrupt context.

The `tcp_v4_do_rcv()` in turn calls other functions for actual TCP processing, depending on the TCP state of the connection. If the connection is in `tcp_established` state, `tcp_rcv_established()` is called; otherwise, `tcp_rcv_state_process()` or other measures would be performed. `tcp_rcv_established()` performs key TCP actions: e.g. sequence number checking, DupACK sending, RTT estimation, ACKing, and data packet processing. Here, we focus on the data packet processing.

In `tcp_rcv_established()`, when a data packet is handled on the **fast path**, it will be checked whether it can be delivered to the `user space` directly, instead of being added to the `receive queue`. The data’s destination in user space is indicated by an `iovec` structure provided to the kernel by the data receiving process through system calls such as `sys_recvmsg`. The conditions of checking whether to deliver the data packet to the user space are as follow:

>- The socket belongs to the currently active process;
>- The current packet is the next in sequence for the socket;
>- The packet will entirely fit into the application-supplied memory location; 

When a data packet is handled on the **slow path** it will be checked whether the data is in sequence (fills in the beginning of a hole in the received stream). Similar to the fast path, an in-sequence packet will be copied to user space if possible; otherwise, it is added to the `receive queue`. Out of sequence packets are added to the socket’s `out-of-sequence queue` and an appropriate TCP response is scheduled. Unlike the `backlog queue`, `prequeue` and `out-of-sequence queue`, packets in the `receive queue` are guaranteed to be in order, already acked, and contain no holes. Packets in `out-of-sequence queue` would be moved to `receive queue` when incoming packets fill the preceding holes in the data stream. The following figure shows the TCP processing flow chart within the interrupt context. 

<img src="{{ site.baseurl }}/images/2015-12-05-1/tcp-interrupt-context-process.png" alt="Networking TCP Processing in Interrupt Context">

As previously mentioned, the backlog and prequeue are generally drained in the process context. The socket’s data receiving process obtains data from the socket through socketrelated receive system calls. For TCP, all such system calls result in the final calling of `tcp_recvmsg()`, which is the top end of the TCP transport receive mechanism. As shown in following figure, when `tcp_recvmsg()` is called, it first locks the socket. Then it checks the receive queue. Since packets in the receive queue are guaranteed in order, acked, and without holes, data in receive queue is copied to user space directly. After that, `tcp_recvmsg()` will process the prequeue and backlog queue, respectively, if they are not empty. Both result in the calling of `tcp_v4_do_rcv()`. Afterward, processing similar to that in the interrupt context is performed. `tcp_recvmsg()` may need to fetch a certain amount of data before it returns to user code; if the required amount is not present, `sk_wait_data()` will be called to put the data receiving process to sleep, waiting for new data to come. The amount of data is set by the data receiving process. Before `tcp_recvmsg()` returns to user space or the data receiving process is put to sleep, the lock on the socket will be released. As shown in Figure 4, when the data receiving process wakes up from the sleep state, it needs to relock the socket again. 

<img src="{{ site.baseurl }}/images/2015-12-05-1/tcp-process-context-process.png" alt="Networking TCP Processing in Process Context">

#### UDP Processing

When a UDP packet arrives from the IP layer, it is passed on to `udp_rcv()`. `udp_rcv()`’s mission is to verify the integrity of the UDP packet and to queue one or more copies for delivery to `multicast` and `broadcast` sockets and exactly one copy to `unicast` sockets. When queuing the received packet in the `receive queue` of the matching socket, if there is insufficient space in the `receive buffer quota` of the socket, the packet may be discarded.
Data within the socket’s `receive buffer` are ready for delivery to the `user space`. 

### Data Receiving Process

Packet data is finally copied from the socket’s `receive buffer` to `user space` by data receiving process through socket-related receive system calls. The receiving process supplies a memory address and number of bytes to be transferred, either in a `struct iovec`, or as two parameters gathered into such a struct by the kernel. As mentioned above, all the TCP socket-related receive system calls result in the final calling of `tcp_recvmsg()`,
which will copy packet data from socket’s buffers (`receive queue`, `prequeue`, `backlog queue`) through `iovec`. For UDP, all the socket-related receiving system calls result in the final calling of `udp_recvmsg()`. When `udp_recvmsg()` is called, data inside receive queue is copied through `iovec` to `user space` directly. 

## Simplified Diagram of the Linux Networking Stack

* http://www.linuxfoundation.org/images/1/1c/Network_data_flow_through_kernel.png

<img src="{{ site.baseurl }}/images/2015-12-05-1/Network_data_flow_through_kernel.png" alt="Networking Data Flow Through Kernel">

## References

This blog entry is mostly an edit based on the following sources, credits should go to these authors!

* http://cd-docdb.fnal.gov/0019/001968/001/Linux-Pkt-Recv-Performance-Analysis-Final.pdf
* http://test-docdb.fnal.gov/0013/001343/001/CHEP06_Talk_132_Wu-Crawford_LinuxNetworking.pdf
* http://www.cs.columbia.edu/~nahum/w6998/papers/2003-wilcox-softirq.pdf
* https://lp007819.wordpress.com/2011/07/21/hardirq-softirqtasklet%E5%92%8Cworkqueue/