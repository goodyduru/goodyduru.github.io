---
layout: post
title: "IPC - Named Pipes"
date: 2023-09-21
categories: os 
---

In the previous article, we talked about what IPC is and listed its different mechanisms. We'll start with one, Named Pipes or FIFO file!

Named pipes are a mechanism that builds upon the structure of anonymous pipes. Anonymous pipes are what we normally call Unix pipes. To understand named pipes, we need to understand anonymous pipes. 

### Anonymous Pipes
Anonymous pipes are memory buffers created and maintained by the kernel. This buffer has two file descriptors for referencing it, one for reading and the other for writing. You can read and write to data to this buffer with the `read` and `write` system calls with the proper descriptors. *Data written to this buffer do not appear on disk*. 

Anonymous pipes are uni-directional i.e you can only write to one end, and read from one end. You can create an anonymous pipe using the [`pipe()`](https://man7.org/linux/man-pages/man2/pipe.2.html) function. Whenever you use the `|` symbol in your shell, it's using the `pipe()` function to create a buffer that your programs can use (along with some redirection tricks that will not be covered here).

One important fact about an anonymous pipe is that *once data is read from the buffer, it is erased from the buffer*.

Imagine you have 2 processes and you want them to communicate without using the shell, how will you do that with anonymous pipes? The simple answer is that you can't. When you call the `pipe()`, the kernel creates the buffer than only that *process and its child processes knows about*. No other process can reference that buffer. If you were to call the `pipe()` function in each process, two separate buffers will be created without any connection between them. This defeats the purpose of your processes communicating. You might ask, how does the shell do it? Well, every process that is executed in your shell is a child of that shell process. Remember what was said about how a pipe buffer can only be known by the creating process and all its child processes :-). That's how your shell is able to get the separate processes to communicate.

We've seen that as cool as anonymous pipes are, there is a major limitation. That limitation is knowledge, not sharing. If one process were to know about another's pipe buffer, it could read and write to that buffer. But, that knowledge is hidden by the kernel. It's isolation by obfuscation! What if we want our process's pipe buffer to be read by other processes without sharing process ancestry? That's what Named Pipes are for!

### Named Pipes
In computing, the first step of allowing access to data stored somewhere is done by creating a reference. That is what Named pipes are, Anonymous pipes plus reference. This reference is simply a file name. This file name is actually stored on disk by your filesystem, and will show up as a file in your explorer. We can test this out by running these commands in your shell.

    mkfifo example-pipes
    ls -l 

The [`mkfifo`](https://man7.org/linux/man-pages/man1/mkfifo.1.html) command creates a named pipe called _example-pipes_. Your output should be similar to this

    prw-r--r--  1 user  group    0 Sep 21 16:36 example-pipes

It's just a file! That's neat. You will notice that the first column starts with `p`. That means it is recognised as a FIFO file. 

Because it's a file, any process can interact with it like every other type of file. That means you can `open` it, `read` from it, `write` to it, `unlink` it, `close` it, and so on. An added bonus about the reference showing up as a file is that file permissions apply to it. This means that you can restrict who has access to it. 

The difference between a normal file and a FIFO file is that no data is stored in it. That will violate the purpose of a pipe which is to act as a communication buffer between processes. The main purpose of the file name is to be a reference, nothing more! Data will not be written to a named pipe until there's at least one concurrent reader.

Named pipes can be bidirectional, meaning that multiple processes can read from it and write to it. All a process with the right permission has to do is to `open` it using the right mode (O_RDWR). It can be tricky to get this right though, because a process can immediately read what it has written, causing the other processes to not see the written data.

### Show me the code
Our example will demonstrate two processes, a server and a client. The client will send `ping` to the client and the server will print it out. Our example will be in Python

Here's the client

```python
    import os

    ROUNDS = 100

    def run():
        pipe_path = '/tmp/ping'
        fd = os.open(pipe_path, os.O_WRONLY)
        i = 0
        while i != ROUNDS:
            os.write(fd, b'ping')
            print("Client: Sent ping")
            i += 1
        os.write(fd, b'end')
        os.close(fd)
```

The named pipe is opened in write-only mode. It sends _ping_ a hundred times to the server using the named pipe called `/tmp/ping`. It signifies that it's done by sending _end_. When it's done, it closes the file. The write will block until there's at least one process that tries to read from the pipe.

Note that data is sent as a byte string and not as a regular string.

Here's the server

```python
    import os

    def run():
        pipe_path = '/tmp/ping'
        os.mkfifo(pipe_path)
        fd = os.open(pipe_path, os.O_RDONLY)
        data = os.read(fd, 4).decode()
        while data != 'end':
            print(f"Server: Received {data}")
            data = os.read(fd, 4).decode()
        os.close(fd)
        os.unlink(pipe_path)
```

Here, a named pipe is created using the [`mkfifo()`](https://man7.org/linux/man-pages/man3/mkfifo.3.html) function. The file is opened in read-only mode. Data is read from the pipe and printed to console in a loop. The loop is ended when _end_ is sent. The file is closed and deleted. The read is blocked until there's data in the pipe.

Note that data has to be decoded to a utf-8 string, because it's originally received as a byte string.

### Performance
Named pipes are pretty fast. [IPC-Bench](https://github.com/goldsborough/ipc-bench#benchmarked-on-intelr-coretm-i5-4590s-cpu--300ghz-running-ubuntu-20041-lts) was able to send 254,880 1KB messages per second on an Intel(R) Core(TM) i5-4590S CPU @ 3.00GHz running Ubuntu 20.04.1 LTS. That's fast enough to serve most processes' communication needs.

### Demo Code
You can find my code that demonstrates unidirectional and bidirectional communication on [Github](https://github.com/goodyduru/ipc-demos).

### Conclusion
Named Pipes is a really simple and powerful IPC mechanism. As with all powerful tools, you have to use it with caution. I wouldn't recommend using it for bi-directional communication, unless you aren't worried about losing data.

This article was filesystem related, the next will be networking related :-). I'm referring to Unix Domain Sockets. Till then, take care of yourself and stay hydrated! ✌🏾
