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
Shared memory is a segment of memory shared between processes. The segment of physical memory is mapped to each processes' virtual memory via their page tables. Once this is done, the process can read and write to this segment just like it does with other segments. Any data written to this segment is visible to other processes. It is possible that this segment's virtual address could differ for each of the participating processes even though they are mapped to the same physical memory segment.

As with [Message Queues](https://goodyduru.github.io/os/2023/11/13/ipc-message-queues.html#message-queues), there are two kinds of Shared Memory - System V and POSIX. This article will focus on System V.

Before creating or accessing a Shared Memory segment, a unique integer key needs to be used. If the key isn't unique, an outsider process might communicate with your own processes which could be annoying or even worse damaging. This key can be hard-coded in each of the participating process, or it could be generated. Hard coding the key reduces the chances of uniqueness because an outsider process could generate one or might have the same key hard coded in it. The key could be generated in a way that guarantees that all participating processes will generate the same key. Fortunately, a function that (almost) guarantees uniqueness and is deterministic exists. This function is called [`ftok`](https://man7.org/linux/man-pages/man3/ftok.3.html). It accepts an existing file path and an integer. As long as the file pointed to by the path isn't recreated, `ftok` will return the same result. All the participating processes need to do is to call the `ftok` function with the same file path and integer and they will get the same key. Extra security steps could include ensuring that the file is only accessible to the participating processes.

Once a key is generated, a process can be use it to get or create a shared memory segment by calling the [`shmget`](https://man7.org/linux/man-pages/man3/shmget.3p.html) function. This function accepts 3 parameters; the key, the size of the shared memory and a flag. If the key isn't associated with a shared memory segment and the **IPC_CREAT** bit is set in the flag, a shared memory segment is created whose size is set to the size parameter . On creating this segment, the function returns an id. If the key has an associated shared memory segment already, an id for the existing segment is returned even if the flag has its **IPC_CREAT** bit set. Errors can happen for several reasons which the man page covers. Permissions can be defined in the flag using the same format as [file permissions](https://www.multacom.com/faq/password_protection/file_permissions.htm).

After successfully getting an id, the virtual memory address of the shared memory segment is gotten by calling the [`shmat`](https://man7.org/linux/man-pages/man3/shmat.3p.html) function. This function accepts 3 parameters; the shared memory id gotten from `shmget`, the virtual memory address the process would like the segment to be mapped and a flag. The OS will pick an address if the address parameter is set to zero. A memory address is returned on successful execution of the function.

After a memory address is returned, a process can communicate with other processes by just reading and writing to the address. There are no special functions for reading and writing to the address. These are done using the same methods as reading and writing to other addresses.

When a process is done communicating, it can detach the segment from its address space by calling the [`shmdt`](https://man7.org/linux/man-pages/man3/shmdt.3p.html) function. The function accepts only one parameter, which is the virtual memory address gotten from `shmat`. Once this function is called, the address becomes invalid and interacting with it could cause a Segmentation Fault.

Detaching the segment does not destroy it, even if all the participating process detach from it. Destroying it is done by calling 

> shmctl(id, IPC_RMID, NULL);

The `id` parameter is the shared memory segment's id gotten from `shmget`, the `IPC_RMID` is a constant that is interpreted by the `shmctl` function to destroy the segment. In addition to destroying a segment, [`shmctl`](https://man7.org/linux/man-pages/man3/shmctl.3p.html) can be used to configure a segment.

A list of all the shared memory segments on a system can be gotten by this command `ipcs -m`.

Enough talk, let's look at an example in Python.

### Show me the code

***

<div id="footer-note-1">[1] Known as Real Mode and Protected Mode in x86.  </div>

***

<div id="footer-note-1">[2] Most CPU architectures support both addressing mode. Booting a system usually starts with physical addressing, after which the kernel switches to virtual addressing before the first application process is even started. You can see how the Linux kernel carries out this switch for x86 <a href="https://github.com/torvalds/linux/blob/9d1694dc91ce7b80bc96d6d8eaf1a1eca668d847/arch/x86/boot/pm.c#L103">here</a>  </div>

***

<div id="footer-note-3">[3] Default size of a page on x86 is 4KB.  </div>

***

<div id="footer-note-4">[4] Almost, because the kernel can prevent a process from accessing some addresses using the <a href="https://unix.stackexchange.com/a/68151">access control bits</a> of a page table entry  </div>

***

<div id="footer-note-5">[5] 256 TiB on x86-64 or 128 PiB with extension which is way bigger than your average RAM size. This massive difference in size between the virtual address size and physical address size allows the combined memory usage of all the processes in your system to be larger than the installed RAM size.   </div>