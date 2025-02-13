---
title: "IPC - Shared Memory"
date: 2024-01-31
tags: [os]
draft: false
---

In the previous article, we covered the [Message Queues](/posts/ipc-message-queues) IPC mechanism. This article will cover another one called Shared Memory.

Before discussing shared memory and how it can be used for IPC, we must know why such a thing as "Shared Memory" exists since we know that all the processes share the RAM they should be sharing memory. The thing is, they share and don't share. This admittedly contradictory event is due to the magic of memory addressing. Let's talk about that for a bit.

### Memory Addressing
There are two modes of addressing memory: __physical__ and __virtual__<sup><a href="#footer-note-1">[1]</a></sup>. In physical addressing mode, when a process reads data from a memory address, the CPU gets it from the same address in RAM and hands it over to the process. Ditto for writes. Physical addressing was the only way to access the RAM in early microprocessors and is still used in many embedded computers.  

Physical addressing is fine when only one process is expected to run on the computer. It is a disaster for a multi-process system. For example, if  2 different processes, A and B, are up simultaneously, A could write to a memory address B also reads from. Even worse, A could write to a memory address containing part of B's executable code even if B is the kernel process.

The above issue is one of the reasons why virtual addressing is the memory address mode used in many microprocessor architectures today<sup><a href="#footer-note-2">[2]</a></sup>. In virtual addressing mode, when a process reads data from a memory address, this memory address is translated to a physical address by the [MMU](https://en.wikipedia.org/wiki/Memory_management_unit), and the data stored at that physical address is fetched from and handed over to the process. This is done without the knowledge of the process. 

This address translation is done using the help of a [__Page Table__](https://en.wikipedia.org/wiki/Page_table). The Page Table stores the mapping between virtual and physical addresses in [page](https://en.wikipedia.org/wiki/Page_(computer_memory))<sup><a href="#footer-note-3">[3]</a></sup> units. Every process has its page table and allows the process to reference __almost<sup><a href="#footer-note-4">[4]</a></sup>__ all the addresses up to the table's maximum address<sup><a href="#footer-note-5">[5]</a></sup>

Virtual addressing allows multiple processes to co-exist without stepping on each other toes. If our 2 processes from the earlier example try to access the same memory address, that address will point to two different addresses in the RAM. So, memory address 4523 could translate to 9619 for A and 86443 for B. This means they don't share virtual memory but share physical memory. 

If processes can only use virtual memory, which cannot be shared, how can they communicate using Shared Memory? Let's look at that.

### Shared Memory
Shared memory is a segment of memory shared between processes. The segment of physical memory is mapped to each process's virtual memory via their page tables. Once this is done, the process can read and write to this segment just like with other segments. Any data written to this segment is visible to other participating processes. The segment's virtual address could differ for each participating process, though it has only one physical address.

As with [Message Queues](/posts/ipc-message-queues#message-queues), there are two kinds of Shared Memory - System V and POSIX. This article will focus on System V.

Creating or accessing a Shared Memory segment requires a unique integer key. If a unique key isn't used, an outsider process might communicate with your processes, which could be annoying or, even worse, damaging. This key can be hardcoded in each of the participating processes, or it could be generated. Hardcoding the key reduces the chances of uniqueness because an outsider process could have the same key hardcoded in it. The key generation should be done in a way that guarantees that all participating processes will reproduce the same key. Fortunately, a function that (almost) guarantees uniqueness and is deterministic exists. This function is called [`ftok`](https://man7.org/linux/man-pages/man3/ftok.3.html). It accepts an existing file path and an integer. As long as the file pointed to by the path isn't recreated, `ftok` will return the same result. All the participating processes need to do is call the `ftok` function with the same file path and integer which will generate the same key. Extra security steps could include ensuring that the file is only accessible to the participating processes.

Once a key is generated, a process can use it to get or create a shared memory segment by calling the [`shmget`](https://man7.org/linux/man-pages/man3/shmget.3p.html) function. This function accepts 3 parameters: the key, the size of the shared memory, and a flag. If the key isn't associated with a shared memory segment and the **IPC_CREAT** bit is set in the flag, a shared memory segment is created whose size is set to the size parameter. On creating a segment, the function returns an id. If the key has an existing shared memory segment, an id for that existing segment is returned even if the flag has its **IPC_CREAT** bit set. Errors can happen for several reasons which, the man page covers. Permissions can be defined in the flag using the same format as [file permissions](https://www.multacom.com/faq/password_protection/file_permissions.htm).

After successfully getting the shared memory id, the virtual memory address of the shared memory segment is returned by the [`shmat`](https://man7.org/linux/man-pages/man3/shmat.3p.html) function. This function accepts 3 parameters: the shared memory id obtained from `shmget`, the virtual memory address the process would like the segment to be mapped, and a flag. The OS will pick an address if the address parameter is set to zero. A memory address is returned on successful execution of the function.

After a memory address is returned, a process can communicate with other processes by just reading and writing to the address. There are no special functions for reading and writing to the address. These are done using the same methods as reading and writing to other addresses.

When a process has finished communicating, it can detach the segment from its address space by calling the [`shmdt`](https://man7.org/linux/man-pages/man3/shmdt.3p.html) function. The function accepts only one parameter, the virtual memory address obtained from `shmat`. Once this function is called, the address becomes invalid, and interacting with it could cause a Segmentation Fault.

Detaching the segment does not destroy it, even if all the participating processes detach from it. Destroying it is done by calling 

> shmctl(id, IPC_RMID, NULL);

The `id` parameter is the shared memory segment's id obtained from `shmget`. The `IPC_RMID` is a constant, meaning the removal of the segment. In addition to destroying a segment, [`shmctl`](https://man7.org/linux/man-pages/man3/shmctl.3p.html) can also be used to configure a segment.

A list of all the shared memory segments on a system can be gotten by this command `ipcs -m`.

Enough talk. Let's take a look at an example in Python.

### Show me the code
This example will demonstrate two processes communicating using a shared memory in Python. Python does not provide out-of-the-box shared memory support. Instead, I made use of the excellent [sysv-ipc](https://semanchuk.com/philip/sysv_ipc/#shared_memory) library. You can find the pip package [here](https://pypi.org/project/sysv-ipc/).

Here's the client code:
```python
import os
import time

import sysv_ipc

ROUNDS = 100

def run():
    path = '/tmp/example'
    fd = os.open(path, flags=os.O_CREAT)
    os.close(fd)
    key = sysv_ipc.ftok(path, 42)
    mem = sysv_ipc.SharedMemory(key, size=20, flags=sysv_ipc.IPC_CREAT, mode=0o644)
    i = 0
    message = b"ping"
    while i != ROUNDS:
        mem.write(message)
        print("Client: Sent ping")
        data = mem.read(4)
        while data == message:
            time.sleep(1e-6)
            data = mem.read(4)
        data = data.decode()
        print(f"Client: Received {data}")
        i += 1
    mem.write(b"end")
    mem.detach()


run()
```
A temp file is (optionally) created to ensure that it exists. The ftok function is called with the file path and an integer. The shared memory segment is (optionally) created and accessed. This returns a shared memory object. A loop is run where a message containing a byte string is sent. To prevent the process from processing a message that it sent, a while loop runs that puts the process to sleep and receives a message. The while loop stops if the message differs from the one sent, signifying that the message was sent from another process. This while loop is a very primitive form of communication synchronization. The message received is decoded and printed, and then the loop continues. The loop is ended when _ROUNDS_ messages are sent. Afterward, an _end_ message is sent to signify that the client is done.

Messages sent and received are byte strings, not regular strings, and thus have to be encoded and decoded accordingly.

Here's the server code

```python
import os
import time

import sysv_ipc

def run():
    path = '/tmp/example'
    fd = os.open(path, flags=os.O_CREAT) # create file
    os.close(fd)
    key = sysv_ipc.ftok(path, 42)
    mem = sysv_ipc.SharedMemory(key, size=20, flags=sysv_ipc.IPC_CREAT, mode=0o644)
    data = mem.read(4)
    message = b"pong"
    while data != b"ping":
        time.sleep(5e-6)
        data = mem.read(4)
    data = data.decode()
    while data[0] != 'e':
        print(f"Server: Received {data}")
        mem.write(message)
        print(f"Server: Sent pong")
        data = mem.read(4)
        while data == message:
            time.sleep(1e-6)
            data = mem.read(4)
        data = data.decode()
    mem.remove()
    mem.detach()
    os.unlink(path)

run()
```

The server code sets up the shared memory segment similarly to the client code. Because `ftok` is called with the exact same parameters as the one in the client code, the generated key value will be the same. A message is sent and a while loop runs that ensures the message received isn't the same as the sent message. A loop is run, which checks that the first character of the received message isn't equal to _e_ which signifies that _end_ wasn't received. Inside this loop, the received message is printed, and a different message is sent. Once sent, another message check runs as a while loop. The message is then received and decoded.

The loop stops once the _end_ message is received. The file for generating the key and the shared memory segment are deleted. The program then exits.

### Performance
Shared message is super fast. There isn't much difference between it and the fastest IPC mechanism (mmap). [IPC-Bench](https://github.com/goldsborough/ipc-bench#benchmarked-on-intelr-coretm-i5-4590s-cpu--300ghz-running-ubuntu-20041-lts) benchmarked 1,659,291 1KB messages per second on an Intel(R) Core(TM) i5-4590S CPU @ 3.00GHz running Ubuntu 20.04.1 LTS. That's super fast.

Due to its blazing-fast speed and its similarity to the standard reading and writing to memory addresses, you'd need to include some form of synchronization mechanism when using Shared Memory just as the code snippets above show.

### Demo Code
You can find my code that demonstrates Shared Memory on [GitHub](https://github.com/goodyduru/ipc-demos).

### Conclusion
Shared Memory is an easy and fast IPC mechanism. It has a straightforward model for sending and receiving messages. Even then, using it comes with the complexity of including a synchronization mechanism. But once you get that right, its speed makes it worth it.

The next article will cover another blazing-fast and familiar IPC mechanism called [Memory-Mapped Files](/posts/ipc-mmap). Till then, take care of yourself and stay hydrated! ✌🏾

***
<ol>
<li><div id="footer-note-1">Known as Real Mode and Protected Mode in x86.  </div></li>

<li><div id="footer-note-2">Most CPU architectures support both addressing modes. Booting this kind of system starts with physical addressing, after which the kernel switches to virtual addressing before the first application process is even started. You can see how the Linux kernel carries out this switch for x86 <a href="https://github.com/torvalds/linux/blob/9d1694dc91ce7b80bc96d6d8eaf1a1eca668d847/arch/x86/boot/pm.c#L103">here</a>  </div></li>

<li><div id="footer-note-3">The default size of a page on x86 is 4KB.  </div></li>

<li><div id="footer-note-4">Almost, because the kernel can prevent a process from accessing some addresses using the <a href="https://unix.stackexchange.com/a/68151">access control bits</a> of a page table entry  </div></li>

<li><div id="footer-note-5">256 TiB on x86-64 or 128 PiB with extension which is way bigger than your average RAM size. This massive difference in size between the virtual and physical address size allows the combined memory usage of all the processes in your system to be larger than the installed RAM size.   </div></li>
</ol>