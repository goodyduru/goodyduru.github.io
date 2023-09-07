---
layout: post
title: "IPC - Introduction"
date: 2023-09-07
categories: operating-systems
---

Have you ever thought of how two processes can communicate in one device? I guess you thought of **files** using the file I/O apis, and yeah that could work, but that would involve some level of complexity in being alerted when data has been written into that file. You could also have thought of **network**, and that works too. But that will involve you selecting ports and setting up all the network shenenigans. So we've eliminated file I/O and network, how else can we make two processses communicate? Before we talk about the how, let's talk about the what. What does it mean for 2 processes to communicate and what is it called?

It's called **Inter Process Communication(IPC)**. You've probably come across this term before. If you have, you've probably wondered about it or ignored it (I've done that too!). Wikipedia defines IPC as *the mechanisms provided by an operating systems for processes to manage shared data*. That's pretty much it! This series of articles will talk about the different mechanisms provided by **Unix-based OSes**. Sorry Windows users! 

Whether or not you've come across IPC, I can bet that you've used them especially if you've worked on a Unix-based OS. When you type the vertical bar symbol(`|`) in your favorite shell program, you are forcing the two processes either side of the symbol to communicate. For example, the below command lists all the text files in our current directory.

```ls -l | grep *.txt```

It works by setting the input of the `grep` process to be the output of the `ls` command. You could say that `ls` is communicating with `grep`. Isn't that neat?!

The above example uses a mechanism called **pipes**, and it's usually the first kind of IPC that most users encounter. While pipes are neat and all, they require the user's intervention to make it happen. If pipes were the only way of communication, you'd need to distribute a shell script that would enable your processes to communicate. Or, you'd need your processes to be children of a parent process and use the `pipe()` system call to set it up. Now, that's not so neat. 

What if you want your processes to be independent and communicate and yet not distribute some arcane shell scripts along with them? Turns out that they are other IPC mechanisms that allow you to do all these. They are:

* Named Pipes
* Unix Domain Sockets
* Unix Signals
* Message Queues
* Shared Memory
* Memory-Mapped Files

Now, files and TCP Sockets (which I called networks) are also IPC mechanisms but I'm not going to talk about those. They are familiar and lots of articles have been written about them. `eventfd` are another mechanism, but they are restricted in the sense that the processes cannot be independent i.e they must share parent process in code.

The next article will be about Named Pipes. Till then, take care of yourself and stay hydrated! ✌🏾