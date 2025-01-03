---
title: "IPC - Memory Mapped Files"
date: 2024-03-18
tags: [os]
draft: false
---

In the previous article, I wrote about the [Shared Memory](/posts/ipc-shared-memory) IPC mechanism. In this article, I will write about a similar IPC mechanism called Memory Mapped Files.

To fully understand this article, you'd need to understand virtual memory, which I covered in the Shared Memory article. Let's talk about how mmap IPC works. <!--more-->

### Memory Mapped IPC
When processes need to communicate with each other via mmap, they use a file as the channel for message transfer. Here's how:

* Each process opens the file using the `open()` function call. This creates the file object in the kernel and returns a file descriptor number.

* The processes call the [`mmap()`](https://man7.org/linux/man-pages/man2/mmap.2.html) function that accepts a starting address, the size, some protection flags, flags (the MAP_SHARED bits must be set for IPC), the file descriptor number from the `open()` call, and starting byte offset as its parameters. The kernel creates the pages in each process's page table whose total size is the specified size argument in the `mmap()` function call. Initially, there is no mapping to any physical address for these pages. The call to the `mmap()` function returns a virtual address to the starting page, which could be the `starting address` argument or an address selected by the OS.

* A page fault happens when the processes initially try to access (read/write) the address returned by `mmap()`. The page fault occurs because there is no mapping from the virtual address to a physical address. This page fault is mitigated when the kernel does the mapping. When the kernel begins mapping, it sees that the page's virtual memory address structure is linked to a file. The kernel checks its [Page Cache](https://github.com/firmianay/Life-long-Learner/blob/master/linux-kernel-development/chapter-16.md) to search for the file page. If it finds the file page, the kernel maps the virtual address to the physical address of the file page in the processes' page tables. If the Page Cache does not contain the file page, the file segment associated with the page is loaded from the disk into the Page Cache, and mapping is done. Once the mapping is complete, the OS resumes the processes, and the address access instruction is re-executed. This mapping happens without the knowledge of the process. This results in a situation where all the various processes' virtual pages are mapped to the same physical pages. This step can happen in each process at different times from other processes, even if communication is simultaneous.

* When a process writes data to an address within the mmaped pages, it is written to the mapped pages in the Page Cache. Since all the other communicating processes have those same pages mmaped into their page tables, they can read that data from that address whether or not it is flushed to disk. Squint and it's like the processes are communicating via a memory buffer mapped into their page tables. Reading and writing to these mmaped addresses is no different from reading and writing to an internal address.

* Unmapping the file is done via the [`munmap()`](https://man7.org/linux/man-pages/man2/mmap.2.html). This removes the mapped pages from the calling process's page table.

Enough theory, let's look at some code.

### Show me the code
Our example will demonstrate two Python processes communicating via mmap: a server and a client. The client will send a "ping" message to the server, and the server will print it out.

Here's the client

```python
import mmap
import os
import time

ROUNDS = 100
FILE_SIZE = 20


def run():
    path = '/tmp/example'
    fd = os.open(path, flags=os.O_CREAT|os.O_RDWR)
    os.lseek(fd, FILE_SIZE, os.SEEK_SET)
    os.write(fd, b" ")
    os.lseek(fd, 0, os.SEEK_SET)
    mem = mmap.mmap(fd, FILE_SIZE)
    i = 0
    message = b"ping"
    while i != ROUNDS:
        mem.write(message)
        print("Client: Sent ping")
        mem.seek(0)
        data = mem.read(4)
        while data == message:
            time.sleep(1e-6)
            mem.seek(0)
            data = mem.read(4)
        data = data.decode()
        print(f"Client: Received {data}")
        mem.seek(0)
        i += 1
    mem.write(b"end")
    mem.close()
    os.close(fd)


run()
```

A temp file is opened (optionally created to ensure it exists). A byte is written to the end of the file to ensure that the file must be as large as the mmaped memory. The file is then mmaped into the client's page table which returns a Python mmap object. A loop runs where the process sends a byte string message. To prevent the process from processing a message that it sent, an inner while loop runs that puts the process to sleep and receives a message. This while loop stops if the message differs from the one sent, signifying that another process sent the message. This while loop is a very primitive form of communication synchronization. The message received is decoded and printed, and then the loop continues. The loop ends when _ROUNDS_ messages are sent. Afterward, the client sends an _end_ message to signify the completion of communication.

The file's current position is reset to the beginning after every read and write. This is because the position is updated to the point after the bytes that were read or written. The program resets this position because it wants to read/write to/from the start of the mmaped address.

Messages sent and received are byte strings, not regular strings, and thus have to be encoded and decoded accordingly.

Here's the server code

```python
import mmap
import os
import time

FILE_SIZE = 20

def run():
    path = '/tmp/example'
    fd = os.open(path, flags=os.O_CREAT|os.O_RDWR) # create file
    os.lseek(fd, FILE_SIZE, os.SEEK_SET)
    os.write(fd, b" ")
    os.lseek(fd, 0, os.SEEK_SET)
    mem = mmap.mmap(fd, FILE_SIZE)
    data = mem.read(4)
    mem.seek(0)
    message = b"pong"
    while data != b"ping":
        time.sleep(5e-6)
        data = mem.read(4)
        mem.seek(0)
    data = data.decode()
    while data[0] != 'e':
        print(f"Server: Received {data}")
        mem.write(message)
        print(f"Server: Sent pong")
        mem.seek(0)
        data = mem.read(4)
        while data == message:
            time.sleep(1e-6)
            mem.seek(0)
            data = mem.read(4)
        data = data.decode()
        mem.seek(0)
    mem.close()
    os.close(fd)
    os.unlink(path)


run()
```

The server code sets up the mmap pages similarly to the client code. It sends a message, and a while loop runs that ensures the message received isn't the same as the sent message. A loop runs checking the first character of the received message isn't equal to _e_ (only "end" will start with 'e'). If it isn't an "end" message, the content of the loop runs as thus: The message is printed to the console, and the program sends a different message. Once sent, another message check runs as an inner while loop. The message is then received and decoded.

The loop stops once the _end_ message is received. The file used for mmap is deleted. The program then exits.

Note that the file position is reset to the start, similar to the client's code.

### Performance
Memory-mapped file IPC is super fast. It is the fastest IPC mechanism in the [IPC-Bench](https://github.com/goldsborough/ipc-bench#benchmarked-on-intelr-coretm-i5-4590s-cpu--300ghz-running-ubuntu-20041-lts) benchmark tests. Its throughput was 1,701,759 msg 1KB messages per second on an Intel(R) Core(TM) i5-4590S CPU @ 3.00GHz running Ubuntu 20.04.1 LTS. That's super fast.

Its speed was a factor that made Cloudflare [switch](https://blog.cloudflare.com/scalable-machine-learning-at-cloudflare) from Unix Sockets to it in their machine learning service.

Due to its blazing-fast speed and its similarity to the standard reading and writing to memory addresses, you'd need to include some type of synchronization mechanism when using mmap files like the code snippets above do.

### Demo Code
You can find my code that demonstrates mmap files on [GitHub](https://github.com/goodyduru/ipc-demos).

### Conclusion
Memory-mapped files are a very fast and versatile IPC mechanism. The only caveat is that you need to include a synchronization mechanism for effective and robust communication. Get this right, and it's worth it.

This brings me to the conclusion of my series on Inter-process communication. Hope you enjoyed them. Thanks for reading. Take care of yourself and stay hydrated! ✌🏾