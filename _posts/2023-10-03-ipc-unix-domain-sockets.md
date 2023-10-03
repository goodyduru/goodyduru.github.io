---
layout: post
title: "IPC - Unix Domain Sockets"
date: 2023-10-03
categories: os
---

In the previous article, We discussed the [Named Pipes](https://goodyduru.github.io/os/2023/09/26/ipc-named-pipes.html) mechanism to achieve Inter-Process Communications. This article will cover another one called Unix Domain Sockets.

Sockets are the Unix abstraction of networking. When we think of networking, we imagine communication. The tools that make up the internet are majorly concerned with creating and maintaining communication pipes between computers. Our Operating Systems provide some of these tools. Since these are communication tools, what if we could use a few of these high-quality and reliable tools provided by our Operating Systems to enable processes to chat with each other? Good news! It turns out that it exists, and that's the subject of this article.

### A Quick Networking Primer
Before we discuss Unix Domain Socket, let's talk about networking quickly. I can assume that we are familiar with the OSI networking layers. I won't list all the layers here because some aren't needed. The layers that are useful for communication are physical, link, network, transport, and application.

The physical layer (optic fiber, 4G, 5G, and others) is concerned with sending our data via a physical medium like radio waves and light. The link layer (Ethernet, WiFi, and others) transfers data between networks. The network layer (BGP, ICMP, and others) is concerned with routing data through the most efficient path. The transport layer (TCP, UDP, and others) transfers data to the **correct application process** on the right computer. The application layer (SMTP, HTTP, ten thousand other protocols) handles the interaction between the user and the data.

When you send data over a network, it goes from application -> transport -> network -> link -> physical. When you receive data over a network, it has gone from physical -> link -> network -> transport -> application.

Inter-process communication mechanisms are responsible for data transfer from one application process to the **correct application process** within a computer. That is very similar to what the transport layer does. Thankfully, our various OSes creators realized this and provided a mechanism that builds upon the transport layer interface. You read that right; Unix Domain Sockets builds upon the Unix transport layer interface. Now, I'm tired of saying transport layer interface. It is quite a mouthful. I will call it BSD Sockets from here on out, as they are popularly known.

### BSD Sockets
BSD/Berkeley/POSIX Sockets is the API provided by Unix-based OSes for the internet. Many of your application programs that communicate over the internet use BSD Sockets, including your favorite web browsers and servers. Many networking libraries used in your application directly or indirectly (via other libraries) use BSD Sockets. The API is so popular and ubiquitous that your favorite programming languages provide a wrapper over the C programming language version.

In addition to being used for the internet, the BSD Sockets API is used for IPC. The API is so well-designed that most of its functions used to achieve network communication are the same as those used for IPC. Let's look at some of these functions and their definitions:

* [`socket()`](https://man7.org/linux/man-pages/man2/socket.2.html): Creates a socket/endpoint for communication. It sets the endpoint communication domain (Internet, IPC, and others) and semantics (TCP, UDP, raw) so that the OS can allocate the right resources to enable our communication. It returns an integer in some languages or an object in other languages.
* [`bind()`](https://man7.org/linux/man-pages/man2/bind.2.html): Used on the server side. It maps a reference with a socket so that other processes in and out of our computer can connect. This reference is an IP address and port number for Internet communication or a file name for IPC.
* [`listen()`](https://man7.org/linux/man-pages/man2/listen.2.html): Used on the server side too. It signifies that our socket is willing to accept incoming connections.
* [`connect()`](https://man7.org/linux/man-pages/man2/connect.2.html): Used on the client side. It associates our socket with a server's reference. It is this map that enables our client to talk to a server. In TCP, this function is also responsible for initiating the [three-way handshake](https://en.wikipedia.org/wiki/Handshake_(computing)#TCP_three-way_handshake) to establish a connection. At the risk of sounding redundant, this reference can be an IP address and port number for Internet communication or a file name for IPC.
* [`accept()`](https://man7.org/linux/man-pages/man2/accept.2.html): Used on the server side. Accepts an incoming connection and creates a new socket for this connection. This socket can be an integer in some languages or an object in others. Afterward, data is sent and received over this new socket.
* [`send()`](https://man7.org/linux/man-pages/man2/send.2.html) and [`recv()`](https://man7.org/linux/man-pages/man2/recv.2.html): These functions are responsible for data transfer between peers. 
* [`close()`](https://man7.org/linux/man-pages/man2/close.2.html): Releases resources allocated to a socket. In TCP, it initiates the [four-way handshake](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_termination) that terminates the connection.

We won't discuss functions like [`getaddrinfo()`](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html), [`freeaddrinfo()`](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html), and others because they aren't necessary for IPC. You can learn more about them on Beej's excellent [guide to networking](https://beej.us/guide/bgnet/html/).

### Unix Domain Sockets
I know it took so long to get here; I had to give context :-). Unix Domain Socket, which I'll call UDS going forward is simply IPC over sockets. When two application processes communicate over UDS, they use the buffers allocated to each socket for message transfer. These buffers are set up by the OS when creating sockets.

The referencing socket on the client process is created by calling the `socket()` function. The server process creates a new referencing socket when it `accept()` a connection. The socket created by calling the `socket()` function on the server process, is **never** used for data transfer. Its sole responsibility is to listen for incoming connections.

An application process that wants other processes to connect to it needs a reference so they can send a connection request to it. Just like in Named Pipes, this reference is a **file name**. And just as in Named Pipes, nothing is ever written to this file. It is used only as a reference. This file is like your regular file; meaning it displays in your directory listing or your file explorer application. Here's how it shows up

    srwxr-xr-x  1 user  staff    0 Sep 29 17:13 udsocket

The `s` in the first column stands for _socket_. An important fact about a socket file is that using the `open()` function to access the file will return an error. A consequence of this is almost all application programs are unable to open the file.

An advantage of using a file as a reference is Unix file access rights and authorization applies to the file. You don't have to create an authentication and authorization over UDS to provide security. You can use the OS file permissions for security.

Each peer socket has a read and write buffer. Message sent from one socket is copied from the write buffer of the sending socket to the read buffer of the receiving buffer. It implies that UDS is a bidirectional IPC mechanism, meaning each peer can read and write messages. An advantage of this setup is that a process can connect and communicate with multiple application processes simultaneously without application-level synchronization. The OS provides this synchronization for free, unlike some other IPC mechanisms!

### Show me the code
Our example will demonstrate two Python processes, a server and a client. The client will send a "ping" message to the server, and the server will print it out.

Here's the client

```python
    import socket

    ROUNDS = 100

    def run():
        server_address = './udsocket'
        sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        sock.connect(server_address)

        print(f"Connecting to {server_address}")
        i = 0
        msg = "ping".encode()
        while i < ROUNDS:
            sock.sendall(msg)
            print("Client: Sent ping")

            data = sock.recv(16)
            print(f"Client: Received {data.decode()}")
            i += 1
        sock.sendall("end".encode())
        sock.close()
```

The client process creates a socket object called `sock`. The socket family, which is the first parameter of the `socket()` function is set to `AF_UNIX` for the OS to create a UDS socket. Other families are `AF_INET` and `AF_INET6` for IPv4 and IPv6 communication used for internet communication.  The socket type(second parameter) is set to `SOCK_STREAM`; this guarantees reliable and two-way communication. This type is also used to specify **TCP** protocol for internet communication. Other types are `SOCK_DGRAM` for UDP and `SOCK_RAW` for raw sockets. 

After the socket object is created by the OS, the client process connects to the server using the file name provided as a reference (_udsocket_ in the code) by calling the `connect()` method. Once the connection is successful, it sends and receives a reply from the server a hundred times using the `sendall` and `recv` methods. Note that this data is a Python byte string and must be encoded and decoded. 

After the loop ends, an "end" message is sent to the server process to signify the conclusion of the transfer. The connection is closed using the `close()` method.

Here's the server code

```python
    import os
    import socket


    def run():
        server_path = './udsocket'
        # Delete if the path does exist
        try:
            os.unlink(server_path)
        except OSError:
            pass

        server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        server.bind(server_path)

        server.listen(1)
        msg = "pong".encode()
        while True:
            connection, address = server.accept()
            print(f"Connection from {address}")
            while True:
                data = connection.recv(16).decode()
                if data != "ping":
                    break
                print(f"Server: Received {data}")
                connection.sendall(msg)
                print(f"Server: Sent pong")
            if data == "end":
                print("Server: Connection shutting down")
                connection.close()
            else:
                print(f"Server Error: received invalid message {data}, shutting down connection")
                connection.close()
```

The server process first deletes the socket file to prevent an "Address already in use" error. It creates a server socket object using the same function call and parameters as the client process. The socket is then bound to the socket file by calling the `bind()` method. After which, it is then set to listening mode using the `listen()` method.

The server then runs a loop; inside this loop, incoming connections are accepted using the `accept()` method. This method returns a new socket object and an address. Data transfer will use this new socket. Once this accepted connection is successful, an inner loop starts running where a message is received using the `recv()` method and checked. If it is "ping", it replies with a "pong" message sent using the `sendall()` method.  If it isn't "ping", the inner loop is ended, and the connection socket is closed.

Messages received and sent are Python byte strings and must be encoded and decoded.

### Performance
UDS are pretty fast, though not as fast as Named Pipes. [IPC-Bench](https://github.com/goldsborough/ipc-bench#benchmarked-on-intelr-coretm-i5-4590s-cpu--300ghz-running-ubuntu-20041-lts) benchmarked 127,582 1KB messages per second on an Intel(R) Core(TM) i5-4590S CPU @ 3.00GHz running Ubuntu 20.04.1 LTS. That's fast enough to serve most processes' communication needs.

### What About Localhost?
Since UDS uses the BSD Sockets API like your networking code, why don't we use TCP/UDP localhost for IPC? Sure, you can use them. An advantage of using TCP/UDP for IPC is that you can use your favorite networking library, instead of fiddling with the Sockets API. So, it's easy and familiar, right? The problem is that you suffer a performance cost.

When you use UDS, your computer knows that communication is local thus, does not carry out the whole handshake and acknowledgments done with TCP. In addition, your data doesn't go through the entire IP stack mechanisms. Not having to do this is a huge performance boost for UDS. Other things are bypassed which I won't explain here, but know that UDS is faster than TCP/UDP localhost. If you're still curious, Robert Watson's excellent [explanation](https://lists.freebsd.org/pipermail/freebsd-performance/2005-February/001143.html) explains more.

### Demo Code
You can find my code on UDS on [GitHub](https://github.com/goodyduru/ipc-demos).

### Conclusion
UDS is a very powerful and reliable IPC mechanism, which is why it's popularly used. Use it if you want your application processes to communicate bi-directionally, and extremely high performance isn't a requirement.

The next article will discuss an old and very limited IPC mechanism called Signal. Till then, take care of yourself and stay hydrated! ✌🏾