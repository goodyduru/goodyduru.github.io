---
layout: post
title: "IPC - Named Pipes"
date: 2023-09-26
categories: os
---

In the previous article, We discussed the Named Pipes mechanism to achieve Inter-Process Communications. This article will cover another one called Unix Domain Sockets.

Sockets are the Unix abstraction of networking. When we think networking, we think communication pipe. The tools that make up the internet are majorly concerned with creating and maintaining communication pipes between computers. Some of these tools are provided by our Operating Systems and some are not. Since these are communication tools, what if we could use a few of these high-quality and reliable tools provided by our Operating Systems to enable processes chat with each other? Good news! It turns out that it exists and that's the subject of this article.

### A Quick Networking Primer
Before we discuss Unix Domain Sockets, let's talk about networking quickly. I can assume that we are familiar with the OSI networking layers. I won't list out all the layers here because some are quite frankly not needed. The layers that are used for communication are physical, link, network, transport and application. 

The physical layer (optic fiber, 4G, 5G, ...) is concerned with sending our data via a physical medium like radio waves and light. The link layer (Ethernet, WiFi, ...) concerns itself with transferring data between networks. The network layer (BGP, ICMP, ...) is concerned with routing data through the most efficient path. The transport layer (TCP, UDP, ...) is responsible for transfering data to the **right application process** on the right computer. The application layer (SMTP, HTTP, ten thousand other protocols) handles the interaction between the user and the data.

When you send data over the network, it goes from application -> transport -> network -> link -> physical. When you receive data over network, it has gone from physical -> link -> network -> transport -> application.

Inter-Process Communication mechanisms are responsible for data transfer from one application process to the **right application process** within a computer. This is very similar to what the transport layer does. Thankfully, our various OSes creators realised this and provided a mechanism that builds upon the transport layer interface. You read that right, Unix Domain Sockets builds upon the Unix transport layer interface. Now, I'm tired of saying transport layer interface, it's quite a mouthful. I will just call it BSD Sockets from here on out which is what they are called.

### BSD Sockets
BSD/Berkeley/POSIX Sockets is the API provided by Unix-based OSes for the internet. Majority of your application programs that communicate over the internet use BSD Sockets including your favorite web browsers and servers. Many networking libraries that you use in your application directly or indirectly (via other libraries) use BSD Sockets. The API is so popular and ubiquitous that your favorite programming languages provides a wrapper over the C programming language version.

In addition to being used for the internet, the BSD Sockets API can be used for IPC. The API is so well-designed that most of the functions that are used to achieve network communication are the same as those used for IPC. Let's look at the functions here and what they are used for

* [`socket()`](https://man7.org/linux/man-pages/man2/socket.2.html): Creates a socket/endpoint for communication. This is where we set the communication domain (Internet, IPC, ...), semantics (TCP, UDP, raw) so that the OS can allocate the right resources to enable our communication. It returns an integer in some languages or an object in other languages.
* [`bind()`](https://man7.org/linux/man-pages/man2/bind.2.html): Used on the server side. It maps a reference with our socket so that other processes in and out of our computer can reach our socket. This reference is an IP address and port number for Internet communication and a file name for IPC.
* [`listen()`](https://man7.org/linux/man-pages/man2/listen.2.html): Used on the server side too. It signifies that our socket is willing to accept incoming connections.
* [`connect()`](https://man7.org/linux/man-pages/man2/connect.2.html): Used on the client side. It associates our socket to a server's reference. It is this map that enables our client to talk a server. In TCP, this function is also reponsible for initiating the [three-way handshake](https://en.wikipedia.org/wiki/Handshake_(computing)#TCP_three-way_handshake) to establish a connection. At the risk of being redundant, this reference is an IP address and port number for Internet communication and a file name for IPC.
* [`accept()`](https://man7.org/linux/man-pages/man2/accept.2.html): Used on the server side. It accepts an incoming connection and creates a new socket for this connection. This socket can be an integer in some languages or an object in other languages. This new socket is used for sending and receiving data.
* [`send()`](https://man7.org/linux/man-pages/man2/send.2.html) and [`recv()`](https://man7.org/linux/man-pages/man2/recv.2.html): These functions are responsible for data transfer between peers. 
* [`close()`](https://man7.org/linux/man-pages/man2/close.2.html): Releases resources allocated to a socket. In TCP, it initiates the [four-way handshake](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_termination) that terminates the connection.

We won't talk about functions like [`getaddrinfo()`](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html), [`freeaddrinfo()`](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html), and others because they aren't necessary for IPC. You can learn more about them over on Beej's excellent [guide to networking](https://beej.us/guide/bgnet/html/).

### Unix Domain Sockets
Sorry it took so long to get here, had to give context :-). Unix Domain Sockets which I'll call UDS from here on is simply IPC over sockets. When two application processes communicate over UDS, they are communicating over the same buffer referenced by different sockets in the different applications. This buffer is set up by the OS when connection is made between the two processes.

The referencing socket on the client side is created by calling the `socket()` function. The referencing socket on the server side is created when it `accept()` a connection. The socket created by calling the `socket()` function on the server side is **never** used for data transfer. It is only used for listening to incoming connections. 

An application process that wants to be communicated with needs to have a reference that other processes can send a connection request to. Just like in Named Pipes, this reference is a **file name**. And just as in Named Pipes, nothing is ever written to this file. It is used only as a reference. This file is like your regular file, meaning that it will show up in your directory listing or your file explorer. Here's how it shows up

    srwxr-xr-x  1 user  staff    0 Sep 29 17:13 udsocket

The `s` in the first column stands for _socket_. An important fact about a socket file is that using the `open()` function to access the file will return an error. A consequence of this is almost all application programs will be unable to open the file.

An advantage of using a file as a reference is Unix file access rights and authorization can be applied to the file. You don't have to create an authentication and authorization over UDS to provide security.  You can just use the OS file permissions for security.

The buffer set up by the OS is bidirectional meaning both peers can read and write to it. In addition, this buffer is only used between one client and one server. It is possible for a server to be simultaneously connected to multiple clients. Conversely, it is possible for a client to be simultaneously connected with multiple servers. Each connection has its own buffer. For example, a server communicating with five clients will transfer data over five separate buffers. An advantage of this is synchronization is provided by the OS when using UDS unlike some other IPC mechanisms where you need to provide synchronization.

### Show me the code
Our example will demonstrate two Python processes; a server and a client. The client will send a "ping" message to the server, and the server will print it out.

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

The client process creates a socket object called `sock`. The socket family which is the first parameter of the `socket()` function is set to `AF_UNIX` specifying to the OS to create a UDS socket. Other families are `AF_INET` and `AF_INET6` for IPv4 and IPv6 communication which are used for internet communication. The socket type which is the second parameter is set to `SOCK_STREAM` that guarantees reliable and two-way communication. This type is also used to specify **TCP** protocol for internet communication. Other types are `SOCK_DGRAM` for UDP and `SOCK_RAW` for raw sockets. 

After this socket object is created, the client process connects to the server using the file name provided as a reference (_udsocket_ in the code) by calling the `connect()` method. Once this connection is established, it sends and receives a reply from the server for a hundred times using the `sendall` and `recv` methods. Note that this data is a Python byte string, so it must be encoded and decode. 

After the loop is done, "end" is sent to signify conclusion of the transfer. The connection is closed using the `close()` method.

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

The server process first deletes the socket file to prevent an "Address already in use" error (We can prevent this error by setting _SO_REUSEADDR_ in the socket options). It creates a server socket object using the same function call and parameters as the client process. The socket is then bound to the socket file by calling the `bind()` method. After which, it is then set to listening mode using the `listen()` method.

The server then runs a loop, where it accepts any incoming connections using the `accept()` method. This method returns a new socket object and an address. It is this new socket object that will be used for data transfer. Once this accepted connection is established, an inner loop is ran where message is received using the `recv()` method and checked. If the message is "ping", it replies with a "pong" message sent using the `sendall()` method. If it isn't ping, the inner loop is ended and the connection socket is closed. 

Messages received and sent are Python byte strings and need to be encoded and decoded as such.

### Performance
UDS are pretty fast, though not as fast as Named Pipes. [IPC-Bench](https://github.com/goldsborough/ipc-bench#benchmarked-on-intelr-coretm-i5-4590s-cpu--300ghz-running-ubuntu-20041-lts) benchmarked 127,582 1KB messages per second on an Intel(R) Core(TM) i5-4590S CPU @ 3.00GHz running Ubuntu 20.04.1 LTS. That's fast enough to serve most processes' communication needs.

### What About Localhost?
Since UDS uses the BSD Sockets API just like your networking code, why don't we just use TCP/UDP localhost for IPC? Sure, you can use them. An advantage of using TCP/UDP is you can use your favorite networking library instead of fiddling about with the Sockets API. So, it's really easy and familiar. The problem is that you suffer a performance cost.

When you use UDS, your computer knows that communication is local and thus, does not carry out the whole handshake and acknowledgements that is done the TCP. In addition, your data doesn't go through the entire IP stack mechanisms. Not having to do this is a huge performance boost for UDS. They are other things that are bypassed that I won't explain here, but know that UDS is faster than TCP/UDP localhost. If you're still curious, Robert Watson's excellent [explanation](https://lists.freebsd.org/pipermail/freebsd-performance/2005-February/001143.html) explains more.

### Demo Code
You can find my code on UDS on [GitHub](https://github.com/goodyduru/ipc-demos).

### Conclusion
UDS is a really powerful and reliable IPC mechanism, which is why it is popularly used for IPC. Use it if you want your application processes to communicate bi-directionally and extremely fast performance isn't a requirement.

The next article will discuss a really old and limited IPC mechanism called Signal. Till then, take care of yourself and stay hydrated! ✌🏾