---
layout: post
title: "IPC - Shared Memory"
date: 2024-01-16
categories: os
---

In the previous article, we covered the [Message Queues](https://goodyduru.github.io/os/2023/11/13/ipc-message-queues.html) IPC mechanism. This article will cover another one called Shared Memory.

Before talking about shared memory and how it can be used for IPC, it's important that we know why such a thing as "Shared Memory" exists since we know that all the processes share the same RAM and therefore they should be sharing memory. The thing is they share and don't share memory. This admittedly contradictory event is due to the magic of memory addressing. Let's talk about that for a bit.

### Memory Addressing
There are two ways of addressing memory, __physical__ and __virtual__<sup><a href="#footer-note-1">[1]</a></sup>. In physical addressing mode; when a process reads data from a memory address, the CPU gets that data from the same address in RAM and hands it over to the process. Ditto for write. Physical addressing was the only way to access the RAM in early microprocessors and is still used in a lot of embedded computers.  

Physical addressing is fine when only one process is expected to run on the computer. It is a disaster for a multi-process system. For example, if there are 2 processes running A and B, A could write to a memory address that is read by B. Even worse, A can write to a memory address containing part of B's executable code even if B is the OS kernel.

The above issue is the reason why virtual addressing is the main memory address type used by processes in a lot of microprocessor architectures today<sup><a href="#footer-note-2">[2]</a></sup>. In virtual addressing mode; when a process reads data from a memory address, this memory address is translated to a physical address by the [MMU](https://en.wikipedia.org/wiki/Memory_management_unit), the data stored at that physical address is fetched from and handed over to the process. This is done without the knowledge of the process. 

This address translation is done using the help of a [__Page Table__](https://en.wikipedia.org/wiki/Page_table). The Page Table stores the mapping between virtual addresses and physical addresses in [page](https://en.wikipedia.org/wiki/Page_(computer_memory))<sup><a href="#footer-note-3">[3]</a></sup> units. Every process has its own page table, and allows the process to reference __almost<sup><a href="#footer-note-4">[4]</a></sup>__ all the addresses up to the table's maximum address<sup><a href="#footer-note-5">[5]</a></sup>

Virtual addressing allows multiple processes to co-exist without stepping on each other toes. If our 2 processes from the earlier example try to access the same memory address, that address will point to two different addresses in the RAM. So, memory address 4523 could translate to 9619 for A and 86443 for B. This means they don't share virtual memory, but share physical memory. 

If processes can only use virtual memory which cannot be shared, how can they communicate using Shared Memory? Let's look at that.

### Shared Memory

***

<div id="footer-note-1">[1] Known as Real Mode and Protected Mode in x86.  </div>

***

<div id="footer-note-1">[2] Most CPU architectures support both addressing mode. Booting a system usually starts with physical addressing, after which the kernel switches to virtual addressing before the first application process is even started. You can see where the Linux kernel carries out this switch for x86 <a href="https://github.com/torvalds/linux/blob/9d1694dc91ce7b80bc96d6d8eaf1a1eca668d847/arch/x86/boot/pm.c#L103">here</a>  </div>

***

<div id="footer-note-3">[3] Default size of a page on x86 is 4KB.  </div>

***

<div id="footer-note-4">[4] Almost, because the kernel can prevent a process from accessing some addresses using the <a href="https://unix.stackexchange.com/a/68151">access control bits</a> of a page table entry  </div>

***

<div id="footer-note-5">[5] 256 TiB on x86-64 or 128 PiB with extension which is way bigger than your average RAM size. This massive difference in size between the virtual address size and physical address size allows the combined memory usage of all the processes in your system to be larger than the installed RAM size.   </div>