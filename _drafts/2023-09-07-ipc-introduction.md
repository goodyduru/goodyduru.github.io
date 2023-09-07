---
layout: post
title: "IPC - Introduction"
date: 2023-09-07
categories: operating-systems
---

Have you ever thought of how two processes can communicate in one device? I guess you thought of **files** using the file I/O APIs, and yeah, that could work, but that would involve some level of complexity in being alerted when writing data into the monitored file. You could also have thought of **networks**, and that works. But that will involve selecting ports and setting up all the network shenanigans. So We've eliminated file I/O and networks. How else can we make two processes communicate? Before we talk about the how, let's talk about the what. What does it mean for two processes to communicate? What is it called?

It's called **Inter-Process Communication(IPC)**. You've probably come across this term before. If you have, you've probably wondered about it or ignored it (I've done that!). Wikipedia defines IPC as "*the mechanisms provided by an operating system for processes to manage shared data*." That's pretty much it! This series of articles will talk about the different mechanisms provided by **Unix-based OSes**. Sorry Windows users! 

Whether or not you've encountered the term, I can bet you've used them especially if you've worked on a Unix-based OS. When you type the vertical bar symbol(`|`) in your favorite shell program, you are dictating that the two processes on either side of the bar communicate. For example, the below command lists all the text files in our current directory.

```ls -l | grep *.txt```

It works by setting the input of the `grep` process to be the output of the `ls` command. You could say that `ls` is communicating with `grep`. Isn't that neat?!

The above example uses a mechanism called **pipes**, and it's usually the first kind of IPC that most users encounter. While pipes are neat and all, they require the user's intervention to make it happen. If pipes were the only way of communication, you'd need to distribute a shell script that will enable your processes to communicate. Or, you'd need your processes to be children of a parent process and use the [pipe()](https://man7.org/linux/man-pages/man2/pipe.2.html) system call to set it up. Now, that's not so neat. 

What if you want your processes to be independent, to communicate, and yet not distribute some arcane shell scripts along with them? It turns out that other IPC mechanisms allow you to do all these. They are:

* Named Pipes
* Unix Domain Sockets
* Unix Signals
* Message Queues
* Shared Memory
* Memory-Mapped Files

Files and TCP Sockets (which I call networks) are also IPC mechanisms, but I won't discuss those. They are familiar, and there are lots of articles about them. `eventfd` is another mechanism, but the processes cannot be independent (they must share a parent process in the code).

The next article will be about Named Pipes. Till then, take care of yourself and stay hydrated! ✌🏾