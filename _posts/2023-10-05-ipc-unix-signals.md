---
layout: post
title: "IPC - Unix Signals"
date: 2023-10-05
categories: os
---

In the previous article, we covered [Unix Socket](https://goodyduru.github.io/os/2023/10/03/ipc-unix-domain-sockets.html) and how it can be used for Inter-Process Communication. This article discusses a different and limited form of IPC.

In the IPC mechanisms we've looked at and most other mechanisms, when an application process sends a message to another, an action is taken by the receiving process depending on the message received. The message will most likely be a byte or group of bytes. These bytes need to be parsed and checked to determine the appropriate action to take. The action to be taken might be calling a function, or executing a program expression. Sometimes, no action needs to be taken due to the application process receiving a message that's not intended for it. Yet, the message needs to be parsed to take no action. How about letting the OS do the parsing to determine if our process needs to ignore? Let's go further; what if we want the OS to execute a function in the application process?

Unix Signal offers this and more. But before we discuss more about it, we need to understand more about Unix processes.

### Processes and Process Group
Open your terminal and type this:

    ps

The above command will print out all the processes that have controlling terminals. Here's mine:

    PID   TTY        TIME    CMD
    10891 ttys044    0:00.74 /bin/zsh -il
    15468 ttys056    0:00.33 /bin/zsh -il
    16837 ttys056    0:00.00 sleep 2000

The above [output](https://www.man7.org/linux/man-pages/man1/ps.1.html) lists the pids, terminal type, CPU time, and the process command with its arguments. The _pid_ is what we are mainly concerned about. Whenever we start a process, it is given a unique _pid_ by the OS. Most times, a new _pid_ is more than the previously assigned _pid_ **during an OS uptime<sup><a href="#footer-note-1">[1]</a></sup>**. In database terminology, _pid_ is akin to the PRIMARY KEY for processes.

Now that we know about _pids_, let's talk about [process group](https://en.wikipedia.org/wiki/Process_group). According to Wikipedia, a process group is a collection of one or more processes. Every process belongs to a process group, and every process group has an id called _pgid_. By default, the `ps` command does not display the _pgid_ of a process. But, we can include it in the `ps` output by typing the command `ps -o "pid,tty,time,command,pgid"`. Here's my terminal output:

    PID   TTY        TIME    CMD          PGID
    10891 ttys044    0:00.87 /bin/zsh -il 10891
    15468 ttys056    0:00.41 /bin/zsh -il 15468
    19198 ttys057    0:00.24 /bin/zsh -il 19198

The eagle-eyed reader will notice that a process's `pgid` is the same as its `pid`; that's by design. In **most cases**  when a new process is started, it belongs to its process group of one member (itself). This process group is assigned an id, conveniently the new process's id. If a new process always creates a new process group with a maximum membership count of one, how can multiple processes belong to one group?  Here's a question. If we start a process, one at a time, and that process has its own group, how do we start multiple processes as a group? The answer is to use a common way to automate system administrative tasks, a shell script. Here's a very simple one:

```sh
    sleep 1000 &
    sleep 500
```

It's pretty straightforward; two sleep processes are started concurrently. One sleeps for a thousand seconds, and the other sleeps for five hundred seconds. Here's the output from my terminal:

    PID   TTY        TIME    CMD          PGID
    ---some processes---
    20648 ttys002    0:00.30 /bin/zsh -il 20648
    20815 ttys002    0:00.01 sh run.sh    20815
    20816 ttys002    0:00.00 sleep 1000   20815
    20817 ttys002    0:00.00 sleep 500    20815
    ---some processes---

You can see that three of the four processes in the output share the same process group. These are the shell process and the two sleep processes. This happened because anytime you run a shell script, it assigns its process group to all its child processes. I think, by now, you have a pretty good idea about process group and how it's created.

Let's move on by looking at a fifty-year-old command/system call.

### `kill`
The basic form of the [`kill`](https://man7.org/linux/man-pages/man1/kill.1.html) command simply does what it means: killing a process. We can kill a process using its _pid_. Here's a `ps` output in my terminal:

    PID   TTY        TIME    CMD
    20608 ttys000    0:00.19 -zsh
    20648 ttys002    0:00.31 /bin/zsh -il
    21195 ttys002    0:00.00 sleep 700
    20820 ttys004    0:00.22 /bin/zsh -il

Here's what it outputs after I kill the sleep process by typing `kill 21195`:

    PID   TTY        TIME    CMD
    20608 ttys000    0:00.19 -zsh
    20648 ttys002    0:00.32 /bin/zsh -il
    20820 ttys004    0:00.23 /bin/zsh -il

As you can see, it's no more; it has been .... killed. The `kill` command can also kill multiple processes together; all you have to do is list the pids of the processes you want to kill. When you type `kill 12983 17838 19983` in your terminal, it kills all the processes whose pids were listed. 

In addition to killing multiple processes whose pids are listed, it's possible to kill all processes in a process group. This can be achieved by setting the pid argument to 0. 

The `kill` command also accepts a number or name prefixed with a hyphen. For now, just think about it as a reason for being killed. Let's look at some examples containing this prefixed number or name and what some of their effects are. I will sequentially start up multiple sleep programs in a shell script and try to kill them slightly differently in a different terminal, starting from the one with the least amount of seconds. Here's the shell script:

    ```sh
        sleep 100
        sleep 200
        sleep 300
        sleep 400
        sleep 500
        sleep 600
    ```

Here's the output in the original terminal after I type `kill -1 21862`

    run.sh: line 1: 21862 Hangup: 1               sleep 100

It's dead, and the next one runs with pid 21880. I'm going to kill it using `kill -4 21880`. Here's the output

    run.sh: line 2: 21880 Illegal instruction: 4  sleep 200

Dead, unto the next one running with pid 22063. I'm going to kill it using `kill -5 22063`, and the output is 

    run.sh: line 3: 22063 Trace/BPT trap: 5       sleep 300

The next one is with pid 22099. I'm killing it with `kill -6 22099`. The output

    run.sh: line 4: 22099 Abort trap: 6           sleep 400

The next one is with pid 22131. I'm killing it with `kill -8 22131`, and the output is

    run.sh: line 5: 22131 Floating point exception: 8   sleep 500

The last one with pid 22163. I'll just simply run `kill 22163`. Here's the output

    run.sh: line 6: 22163 Terminated: 15          sleep 600

You can see that for every one of the sleep processes, its reason for being killed is different. Each one of the reasons has two parts: a string and a number. The string output is for human consumption, but the number is more important. If you look carefully at the numbers, you'll see that they correspond with the prefixed number in our kill command except for the last one. It turns out that when you run `kill pid`, you're really running `kill -15 pid`.

One more thing. You can replace those prefixed numbers with prefixed strings. These strings are unique and are understood by the kill command. Each string value is mapped to a number, so running `kill -special_string pid` is the same as running `kill -special_number pid`. For example, `kill -fpe pid` is the equivalent of `kill -8 pid`. They will both result in a _floating point exception_, resulting in the termination of the process. 

By now, I bet you're curious about what those prefixed numbers or strings represent. Don't worry, I've got you :-). They are called **signals**. Let's dive into them.

### Signals
Signals are standardized messages sent to a process by the Operating System. The list of these messages is very limited in number and are defined in every modern POSIX-compliant system. Some OS might have more and some less, but some universal ones are on all UNIX-based OS. Here's a list of them and their meaning on [Wikipedia](https://en.wikipedia.org/wiki/Signal_(IPC)#POSIX_signals).

These messages have a high priority, and thus, the process must be interrupted from its normal flow to handle it. The main reason why they have high priority is because lots of process errors are delivered to a process using signals. For example, you can see from the `kill` output above that some reasons look like error messages, even though the process didn't have an error.

Now, I know 2 ways a signal can be sent to a process. They are [`raise`](https://man7.org/linux/man-pages/man3/raise.3.html) function and `kill` command/syscall. There might be other ways, but I'm ignorant about them. The `raise` function is just a process sending a signal to itself. We've looked at the `kill` command, and I'll talk about its equivalent syscall function soon. The OS kernel may also send a signal by manipulating the process [struct directly](https://github.com/torvalds/linux/blob/7de25c855b63453826ef678420831f98331d85fd/kernel/signal.c#L1722).

Every signal must have a handler function in a process. This function is executed whenever the process receives the signal. The function can be defined in kernel or user-level code. When the OS starts a new application process, it assigns default handlers for every of its signal objects. Some signals' default handlers terminate the process. Some other default handlers don't execute anything, i.e. the signal is ignored. A signal's default handler can be referenced with the constant `SIG_DFL`. You can see a list of default signals' actions [here](https://elixir.bootlin.com/linux/latest/source/include/linux/signal.h#L352).

A signal default handler can be changed to a different handler. This handler can be a defined function or `SIG_IGN`. `SIG_IGN` tells the process to ignore the signal. We set a signal handler by using either the [`signal()`](https://man7.org/linux/man-pages/man2/signal.2.html) or [`sigaction`](https://man7.org/linux/man-pages/man2/sigaction.2.html) functions. We might want to handle a signal once and reset the default handler immediately after. We can do this by setting the signal's handler function to `SIG_DFL` inside our defined handler.

Enough talk; let's demonstrate a simple example. Here's a simple Python script that executes an infinite loop:

```python
    import signal

    def fpe_bulletproof(signum, frame):
        print("You can't kill me, I'm bulletproof")


    def run():
        signal.signal(signal.SIGFPE, fpe_bulletproof)
        while True: pass

    run()
```

Run this script in a terminal. Open a second terminal and look up the pid of the script process using `ps`. Run `kill -8 script_pid` or `kill -fpe script_pid` in the second terminal. Now, go to the first terminal running the script process; you should see the "You can't kill me, I'm bulletproof" printed in the console. What happened is that we've replaced the default handler with the `fpe_bulletproof` function. Every time you run the `kill -8 ...` command, the handler function is executed, and the statement is printed to your terminal console. Now try to run `kill -1 script_pid`, you will see that the script has been terminated. This is due to only setting a handler for `SIGFPE` and not the other signals. 

Note that the handler function must have two parameters: the signum and a frame.

The handlers for almost all signals can be changed, except for `SIGKILL` and `SIGSTOP` signals. These signals cannot be stopped or ignored. You can try this by changing `SIGFPE` to `SIGKILL` and then rerun the script. An "Invalid Argument" exception is thrown. This is why when you want to force-kill a process from the terminal, you type `kill -9 process_pid`. The _9_ represents the `SIGKILL` signal.

A note of warning, you have to be careful of what is executed inside your handler function. Functions called directly and indirectly by your handler function have to be async-safe. Calling async-unsafe functions in your handler function can invoke undefined behavior. A list of async-safe functions can be found [here](https://man7.org/linux/man-pages/man7/signal-safety.7.html).

### How does this concern IPC?
I know, I know. How does this all concern IPC? Here's how, whenever the `kill pid` is run in a terminal, a `kill` process is started. This process sends a signal to the process whose pid is included in the command argument. If you squint a little, it looks like the `kill` process sent a message to the other process. Isn't that Inter-process communication? Can we have the power that `kill` has?

Luckily for us, we can do what kill does. That's because the `kill` process calls the [`kill()`](https://man7.org/linux/man-pages/man2/kill.2.html) system call. We can send a message to another application process by executing the syscall with its _pid_ and a signal.

Carrying out uni-directional communication from one independent process to another is easy when using `kill()`. All we have to do is run the recipient process, get its pid, and then run the sending process with the pid. Letting both processes know each other pids requires other IPC mechanisms to communicate their pids. What if we don't want to do that? What if we want both processes to call the `kill` function without knowing each other's pids?

We can answer the above questions with two words "process groups". We can start our processes with the same process groups and send signals to each other using `kill(0, signum)`. With this, there's no need for pid exchange, and IPC can be carried out in blissful pids ignorance.

### Show me the code
Here's a demonstration of two Python processes communicating with Signals. Here's the first:

```python
    import os
    import signal

    i = 0
    ROUNDS = 100

    def print_pong(signum, frame):
        global i
        os.write(1, b"Client: pong\n")
        i += 1


    def run():
        signal.signal(signal.SIGUSR1, print_pong)
        while i < ROUNDS:
            os.kill(0, signal.SIGUSR2)
        os.kill(0, signal.SIGQUIT)

    run()
```

The above code sets a handler function for the `SIGUSR1` signal. This function prints a statement to the console using `os.write` and increment i. We use `write` rather than `print` because `print` isn't async-safe. It then sends the `SIGUSR2` signal in a while loop. When it receives a hundred `SIGUSR1` signals the loop ends, and a `SIGQUIT` signal is sent. 

Here's the second process:

```python
    import os
    import signal

    should_end = False

    def end(signum, frame):
        global should_end
        os.write(1, b"End\n")
        should_end = True

    def print_ping(signum, frame):
        os.write(1, b"Server: ping\n")
        os.kill(0, signal.SIGUSR1)


    def run():
        signal.signal(signal.SIGUSR2, handler=print_ping)
        signal.signal(signal.SIGQUIT, end)

        while not should_end:
            pass
    
    run()
```

This one handles 2 signals, `SIGUSR2` and `SIGQUIT`. Each of these signals has its handler function. On receiving a `SIGUSR2` signal, the process prints out a statement and sends a `SIGUSR` signal. When it receives the `SIGQUIT` signal, the process sets the boolean variable `should_end` to True. This will end the infinite loop and ensure our program exits.

It is possible to have one handler function for several signals. We can handle each signal differently based on the value of the `signum`.

You'll notice that both programs set the pid parameter of `os.kill` to 0. This works because both processes run in the same process group. Here's the shell script that is used to run the processes:

```sh
    trap "" USR1 USR2 QUIT
    python3 server.py & python3 client.py
```

We use the trap instruction to handle all the signals sent by both Python processes. This is because `kill(0, sig)` sends a signal to all the processes in a process group, and the shell process is in the same process group with its default handler(termination). We don't want that, and that's why we handle them with an empty statement.

### Performance
Signals are plenty fast. [Cloudflare](https://blog.cloudflare.com/scalable-machine-learning-at-cloudflare/#ipc-mechanisms) benchmarked 	404,844 messages per second<sup><a href="#footer-note-2">[2]</a></sup>. That can suit most performance needs.

### Demo Code
You can find my code that demonstrates Unix Signals on [GitHub](https://github.com/goodyduru/ipc-demos).

### Conclusion
Unix signals are a straightforward but limited mechanism for IPC. They can do much more than IPC, e.g. set alarms, handling errors. There are some issues in using it, so be cautious.

The next article will cover a mechanism I didn't know existed until recently called [Message Queues](https://goodyduru.github.io/os/2023/11/13/ipc-message-queues.html). Till then, take care of yourself and stay hydrated! ✌🏾

***

<div id="footer-note-1">[1] It is possible for pids to hit the maximum limit during uptime, thus causing a wrap around to a smaller <a href="https://superuser.com/a/135008">value</a>. Some Unix-based OSes give you the ability to enable <a href="https://security.stackexchange.com/a/89961">random PID generation</a>  </div>

***

<div id="footer-note-2">[2] The table shows performance for sending 1KB messages. This is misleading for Signals. No data is sent via this mechanism. </div>