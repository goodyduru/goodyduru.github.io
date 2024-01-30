---
layout: post
title: "IPC - Message Queues"
date: 2023-11-13
categories: os
---

In the previous article, we covered [Unix Signal](https://goodyduru.github.io/os/2023/10/05/ipc-unix-signals.html) and its usage for IPC. An implementation of a popular asynchronous communication pattern will be covered in this article.

### Message Queues 
There are two types of message queues - System V and POSIX. There are lots of similarities between them, with minor differences. This article focuses on System V because it is the more widely supported type. 

In basic terms, a [message queue](https://github.com/torvalds/linux/blob/6bc986ab839c844e78a2333a02e55f02c9e57935/ipc/msg.c#L49) is a linked list of messages. The OS can maintain several lists of sent messages, each referenced by a unique integer identifier. A message is sent by appending to the list and received by popping from the head of the list. This list is managed by the OS kernel and is stored in memory. In-memory storage of the list allows for asynchronous communication. This means communicating processes do not need to be interacting with the same queue at the same time.

Before creating or accessing a message queue, a unique key is to be generated deterministically. It needs to be unique to avoid an error when creating a queue. The same key must be used by all the application processes to communicate via the same queue. A recommended way for generating a key is to call the [`ftok`](https://man7.org/linux/man-pages/man3/ftok.3.html) function. This function accepts a file path and an integer. The file path must be an existing file or an error will be returned. A recommended file path can be the application configuration file. The `ftok` function will always return the same result as long as the file isn't deleted and recreated. Collisions can happen, but the chances of those are pretty slim.

Accessing or creating a message queue is done using the [`msgget`](https://man7.org/linux/man-pages/man2/msgget.2.html) function. This function accepts a key and a flag parameter. Creating a message queue is done by specifying **IPC_CREAT** in the flag. The message queue permission is defined in the flag parameter. This permission has the same format as [file permissions](https://www.multacom.com/faq/password_protection/file_permissions.htm). For example, creating a message queue that grants only the effective user write permission, and read permission to others is done by executing `msgget(key, IPC_CREAT | 0644)`. This creates a message queue where a process with its userid not set to the queue owner's id can receive a message but cannot send one. The `msgget` function returns the message queue identifier, which is used when sending and receiving messages from the queue.

You can view all the message queues in your OS by running `ipcs -q`.

Fundamentally, messages are sent and received as bytes using the [`msgsnd` and `msgrcv`](https://man7.org/linux/man-pages/man2/msgsnd.2.html) functions. The `msgsnd` function accepts a message queue id returned by the `msgget` function, a pointer to the message struct, the size of the message, and an integer flag. The message struct needs to include an integer message type. The `msgsnd` function appends the message to the queue identified by the id specified in its first parameter. 

Message queues have a maximum size specified by the OS, which can be [configured](https://www.ibm.com/docs/en/db2/11.1?topic=unix-modifying-kernel-parameters-linux) by the user. This means that `msgsnd` will block if the queue is full. Preventing blocking can be done by specifying `IPC_NOWAIT` in the flag argument, which will return an error instead.

Receiving a message is done using the `msgrcv` function. The difference between these function parameters and `msgsnd` parameters is the message type. This type determines if the process wants to read any message (0), a specific message type (a positive integer), or a particular message group (a negative integer). Setting the type to a negative integer reads any message whose type is less than or equal to the absolute value of the specified type. This function removes a message from the queue and copies it to the provided message buffer parameter.

The `msgrcv` function will by default block if the queue is empty/no message of the specified type(s). This can be prevented by specifying `IPC_NOWAIT` in the flag argument, which makes the function return an error. In Linux, if there's no message of the type(s), the first message in the queue can be read if `MSG_EXCEPT` is specified in the flag argument.

Message queues can be removed and configured by the [`msgctl`](https://man7.org/linux/man-pages/man2/msgctl.2.html) function. The function also allows for reading a message queue metadata.

### Show me the code
This example will demonstrate two processes communicating using a message queue in Python. Unfortunately, Python does not provide out-of-the-box message queue support. Instead, I made use of the excellent [sysv-ipc](https://semanchuk.com/philip/sysv_ipc/#message_queue) library. You can find the pip package [here](https://pypi.org/project/sysv-ipc/).

Here's the client code:
```python
    import os

    import sysv_ipc

    ROUNDS = 100


    def run():
        path = '/tmp/example'
        fd = os.open(path, flags=os.O_CREAT)
        os.close(fd)
        key = sysv_ipc.ftok(path, 42)
        mq = sysv_ipc.MessageQueue(key, flags=sysv_ipc.IPC_CREAT, mode=0o644)
        msg_type = 10
        i = 0
        while i != ROUNDS:
            mq.send(b"ping", type=msg_type)
            print("Client: Sent ping")
            data, _ = mq.receive(type=(msg_type+1))
            data = data.decode()
            print(f"Client: Received {data}")
            i += 1
        mq.send(b"end", type=msg_type)


    run()
```
A temp file is (optionally) created to ensure that it exists. The ftok function is called with the file path and an integer. The message queue is (optionally) created and accessed. This returns a message queue object. A loop is run where a message containing a byte string and its type set to *msg_type* is sent. A message of a different type is then received. A different message type has to be specified to prevent a process from receiving a message sent by it. The message received is decoded and printed, and then the loop continues. The loop is ended when _ROUNDS_ messages are sent. Afterward, an _end_ message is sent to signify that the client is done.

Messages sent and received are byte strings, not regular strings, and thus have to be encoded and decoded accordingly.

Here's the server code

```python
    import os

    import sysv_ipc

    def run():
        path = '/tmp/example'
        fd = os.open(path, flags=os.O_CREAT) # create file
        os.close(fd)
        key = sysv_ipc.ftok(path, 42)
        mq = sysv_ipc.MessageQueue(key, flags=sysv_ipc.IPC_CREAT, mode=0o644)
        msg_type = 10
        data, _ = mq.receive(type=msg_type)
        data = data.decode()
        while data != 'end':
            print(f"Server: Received {data}")
            mq.send(b"pong", type=(msg_type+1))
            print(f"Server: Sent pong")
            data, _ = mq.receive(type=msg_type)
            data = data.decode()
        os.unlink(path)
        mq.remove()


    run()
```

The server code sets up the message queue similarly to the client code. Because `ftok` is called with the exact parameters as the one in the client code, the generated key value will be the same. A message of the type *msg_type* is received from the message queue. A loop is run, which checks that the received message isn't equal to _end_. Inside this loop, the received message is printed, and a message of a different type is sent to the queue. The message is then received and decoded.

The loop stops once the _end_ message is received. The file for generating the key and the message queue are deleted. The program then exits.

### Performance
Message queues are pretty fast. [IPC-Bench](https://github.com/goldsborough/ipc-bench#benchmarked-on-intelr-coretm-i5-4590s-cpu--300ghz-running-ubuntu-20041-lts) benchmarked 213,796 1KB messages per second on an Intel(R) Core(TM) i5-4590S CPU @ 3.00GHz running Ubuntu 20.04.1 LTS. That's fast enough for most inter-process communication needs.

### Demo Code
You can find my code that demonstrates Message Queues on [GitHub](https://github.com/goodyduru/ipc-demos).

### Conclusion
Message Queues are a familiar communication pattern. Using them for your unidirectional or bi-directional inter-process communication needs is a good choice.

The next article will cover a blazing-fast IPC mechanism called Shared Memory. Till then, take care of yourself and stay hydrated! ✌🏾
