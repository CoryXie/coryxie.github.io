---
layout: page
permalink: /about/index.html
title: Cory Xie
tags: [CoryXie]
imagefeature: fourseasons.jpg
chart: true
---
<figure>
  <img src="{{ site.baseurl }}/images/CoryXie.png" alt="Cory Xie">
  <figcaption>Cory Xie</figcaption>
</figure>

{% assign total_words = 0 %}
{% assign total_readtime = 0 %}
{% assign featuredcount = 0 %}
{% assign statuscount = 0 %}

{% for post in site.posts %}
    {% assign post_words = post.content | strip_html | number_of_words %}
    {% assign readtime = post_words | append: '.0' | divided_by:200 %}
    {% assign total_words = total_words | plus: post_words %}
    {% assign total_readtime = total_readtime | plus: readtime %}
    {% if post.featured %}
    {% assign featuredcount = featuredcount | plus: 1 %}
    {% endif %}
{% endfor %}


My name is **Cory Xie**, and this is my personal blog. It currently has {{ site.posts | size }} posts in {{ site.categories | size }} categories which combinedly have {{ total_words }} words, which will take an average reader ({{ site.wpm }}WPM) approximately <span class="time">{{ total_readtime }}</span> minutes to read. {% if featuredcount != 0 %}There are <a href="{{ site.baseurl }}/featured">{{ featuredcount }} featured posts</a>, you should definitely check those out.{% endif %} The most recent post is {% for post in site.posts limit:1 %}{% if post.description %}<a href="{{ site.baseurl }}{{ post.url }}" title="{{ post.description }}">"{{ post.title }}"</a>{% else %}<a href="{{ site.baseurl }}{{ post.url }}" title="{{ post.description }}" title="Read more about {{ post.title }}">"{{ post.title }}"</a>{% endif %}{% endfor %} which was published on {% for post in site.posts limit:1 %}{% assign modifiedtime = post.modified | date: "%Y%m%d" %}{% assign posttime = post.date | date: "%Y%m%d" %}<time datetime="{{ post.date | date_to_xmlschema }}" class="post-time">{{ post.date | date: "%d %b %Y" }}</time>{% if post.modified %}{% if modifiedtime != posttime %} and last modified on <time datetime="{{ post.modified | date: "%Y-%m-%d" }}" itemprop="dateModified">{{ post.modified | date: "%d %b %Y" }}</time>{% endif %}{% endif %}{% endfor %}. The last commit was on {{ site.time | date: "%A, %d %b %Y" }} at {{ site.time | date: "%I:%M %p" }} [UTC](http://en.wikipedia.org/wiki/Coordinated_Universal_Time "Temps Universel Coordonné").

KEY CAPABILITIES
================

The following list shows the key technical aspects that either I am quite expert at or have enough know-hows.

-   Hands on Technical Domains: Bootloader, BSP, OS Kernel, Device Drivers and Driver Frameworks, Application Frameworks

-   Mainstream operating systems: Linux (Android), VxWorks, FreeBSD, ThreadX,…

-   Mainstream CPU architectures: Intel x86, Intel x64, ARM, PowerPC, MIPS,…

-   Common CPU architecture knowledge: Cache Coherency (MESI, MESIF), Memory Consistence (Memory Ordering, Write Buffers), MMU (Segmentation, Paging, TLB), Virtualization (VT-x, VT-d),…

-   Common protocols and standards: USB 2.0/3.0 (xHCI, EHCI, OHCI, UHCI,…), PCI/PCIe (RC, Bridge, Switch,…), SATA (AHCI), SCSI (USB MSC, iSCSI), RAID, IPv4/IPv6 (Linux Stack), Bluetooth, ZigBee,…

-   Common devices and interfaces: Ethernet, NOR/NAND (MTD, FTL,…), SSD, DDR, I, SPI, UART,…

-   Mainstream development languages: ASM (ARM, X86, PowerPC, MIPS), C, C++, Java, Bash, Python,…

-   Common hardware debugging tools: Oscilloscopes, Logic analyzers, USB analyzers, ICE tools,…

-   Frequently used development tools: GNU Linux (bash, make, autotools,…), Git, Clear Case, Eclipse, IDA Pro,…

-   Mainstream development processes: Waterfall, Prototyping, Agile,…

SELF ACCESSMENT
===============

I am truly a technical person, with a very strong intention to gain “full stack expertise” in the field of System Software. I have strong interests in low level technologies (as in how Cache Coherency works) that enable high level technologies (as in how Internet-of-Things works). With these low level knowledge bases as well as enough engineering experiences, I believe I could achieve my career goals. However, technology is not everything that defines me. In fact, I would expect the following personal traits would finally define me and my career.

-   Positive, sincere, well-motivated, and open-minded.

-   Can endure hardship, with a strong sense of responsibility.

-   Have a strong ability for self-learning and independent analysis.

-   Good at system-level thinking, with a strong spirit in team working.

