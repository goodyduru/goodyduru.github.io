---
layout: post
title: "IPC - Unix Signals"
date: 2023-10-05
categories: os
---

In the previous article, we covered [Unix Sockets](https://goodyduru.github.io/os/2023/10/03/ipc-unix-domain-sockets.html) and how it can be used for Inter-Process Communication. This article discusses a different and limited form of IPC.

In the IPC mechanisms we've looked at and most other mechanisms, when an application process sends a message to another, an action is taken by the receiving process depending on the message received. The message will most likely be a byte or group of bytes. These bytes need to be parsed and checked to determine the appropriate action to take. The action to be taken might be calling a function, or executing a program expression. Sometimes, no action needs to be taken due to the process receiving a message that's not intended for it. Yet, the message needs to be parsed to take no action. How about if we let the OS do the parsing to determine if our process needs to ignore? Let's go further, what if we want the OS to execute a function in the application process?

Unix Signals offers this and more. But before we discuss more about it, we need to understand more about Unix processes.

### Processes and Process Group
Open your terminal and type this:

    ps

The above command will print out all the processes that have controlling terminals. Here's mine:

    PID   TTY        TIME    CMD
    10891 ttys044    0:00.74 /bin/zsh -il
    15468 ttys056    0:00.33 /bin/zsh -il
    16837 ttys056    0:00.00 sleep 2000

The above [output](https://www.man7.org/linux/man-pages/man1/ps.1.html) lists out the pids, terminal type, CPU time and the process command with its arguments. The _pid_ is what we are mainly concerned about. Whenever we start a process, it is given a unique _pid_ by the OS. A new _pid_ is always greater than the previously assigned _pid_ **during an OS uptime<sup><a href="#footer-note-1">[1]</a></sup>**. In database terminology, _pid_ is akin to the PRIMARY KEY for processes. There are special pids which are never assigned to a process, but they have special significance. We will talk about them later.

Now that we know about _pids_, let's talk about [process group](https://en.wikipedia.org/wiki/Process_group). According to Wikipedia, a process group is a collection of one or processes. Every process belongs to a process group and every process group has an id called _pgid_. By default, the `ps` command does not display the _pgid_ of a process. But, we can display that by typing the command `ps -o "pid,tty,time,command,pgid"`. Here's what mine displays:

    PID   TTY        TIME    CMD          PGID
    10891 ttys044    0:00.87 /bin/zsh -il 10891
    15468 ttys056    0:00.41 /bin/zsh -il 15468
    19198 ttys057    0:00.24 /bin/zsh -il 19198

The eagle-eyed reader will notice that a process's `pgid` is the same as its `pid`, that's by design. You see, in **most cases**  when a new process is started, it belongs to its own process group of one member (itself). This process group is assigned an id, which is conveniently the new process's id. If a new process always creates a new process group, which has a maximum membership count of one, how can multiple processes belong to one group? Here's a hint. If we start a process, one at a time in the terminal and that process has its own group; how do we start multiple processes as a group? The answer is use a a common way to automate system administrative task, a shell script. Here's a very simple one:

```sh
    sleep 1000 &
    sleep 500
```

It's pretty straightforward, two sleep processes are started concurrently. One sleeps for a thousand seconds, the other sleeps for five hundred seconds. Here's the output from my terminal:

    PID   TTY        TIME    CMD          PGID
    ---some processes---
    20648 ttys002    0:00.30 /bin/zsh -il 20648
    20815 ttys002    0:00.01 sh run.sh    20815
    20816 ttys002    0:00.00 sleep 1000   20815
    20817 ttys002    0:00.00 sleep 500    20815
    ---some processes---

You can see that three out of the four processes in the output share the same process group, the shell process and the two sleep processes. That's because anytime you run a shell script, it assigns its process group to all its child processes. I think by now, you have a pretty good idea about process group and how its assigned. 

Let's move on by taking a look at a fifty year old command/system call.

### `kill`
The basic form of the [`kill`](https://man7.org/linux/man-pages/man1/kill.1.html) command simply does what it means, it kills a process. We can kill a process using its _pid_. Here's a `ps` output in my terminal:

    PID   TTY        TIME    CMD
    20608 ttys000    0:00.19 -zsh
    20648 ttys002    0:00.31 /bin/zsh -il
    21195 ttys002    0:00.00 sleep 700
    20820 ttys004    0:00.22 /bin/zsh -il

Here's what it outputs after I kill the sleep process, by typing `kill 21195`:

    PID   TTY        TIME    CMD
    20608 ttys000    0:00.19 -zsh
    20648 ttys002    0:00.32 /bin/zsh -il
    20820 ttys004    0:00.23 /bin/zsh -il

As you can see, it's no more; it has been .... killed. The `kill` command can also kill multiple process at the same time; all you have to do is to list out the pids of the processes you want to kill. When you type `kill 12983 17838 19983` in your terminal, it kills all the processes whose pids were listed. 

In addition to killing a multiple processes whose pids are listed, it is also possible to kill all processes in a process group. This can be achieved by setting the pid argument to 0. 

The `kill` command also accepts a special number or name. This special number or name must be prefixed with a `-`. For now, just think about it as a reason to being killed. Let's take a look at some examples containing this special number or name and what their effects are. I will sequentially start up multiple sleep programs in a shell script and try to kill them a little differently in a different terminal starting from the one with the least seconds. Here's the shell script:

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

It's dead and the next one runs with pid 21880. I'm going to kill it using `kill -4 21880`. Here's the output

    run.sh: line 2: 21880 Illegal instruction: 4  sleep 200

Dead, unto the next one which runs with pid 22063. I'm going to kill it using `kill -5 22063` and the output is 

    run.sh: line 3: 22063 Trace/BPT trap: 5       sleep 300

Next one with pid 22099. I'm killing it with `kill -6 22099`. The output

    run.sh: line 4: 22099 Abort trap: 6           sleep 400

The next one with pid 22131. I'm killing it with `kill -8 22131` and the output is

    run.sh: line 5: 22131 Floating point exception: 8   sleep 500

The last one with pid 22163. I'll just simply type `kill 22163`. Here's the output

    run.sh: line 6: 22163 Terminated: 15          sleep 600

You can see that for each of the sleep processes, the reason for being killed is different. Each reason has two parts, a string and a number. The string is for human consumption, but the number is what is special. If you look carefully at the numbers, you'd see that they correspond with the prefixed number in our kill command except for the last one. It turns out that when you run `kill pid`, you're actually running `kill -15 pid`.

One more thing, you can replace those prefixed numbers with prefixed strings. These strings are unique and are understood by the kill command. Each number is mapped to a string and so running `kill -special_string pid` is the same as running `kill -special_number pid`. For example, `kill -fpe pid` is the equivalent of `kill -8 pid`. They will both result in a _floating point exception_ and the process will terminate. 

By now, I bet you're curious about what those special numbers or strings are. Don't worry, I've got you :-). They are called **signals**. Let's talk about them

### Signals
Signals are simply standardised messages that are sent to a process by the Operating System. These messages are very limited in number and are defined in every modern POSIX compliant system. Some OS might have more and some less, but there are some universal ones which are on all UNIX-based OS. Here's a list of them and their meaning on [Wikipedia](https://en.wikipedia.org/wiki/Signal_(IPC)#POSIX_signals).

These messages have a very high priority and thus, the process must be interrupted from its normal flow in order to handle it. The reason why they have high priority is because lots of process errors are delivered to a process using signals. For example, you can see from the `kill` output above that some of the reasons look like error messages, even though the process didn't have an error.

Now, I know of 2 ways by which a signal can be sent to a process. They are [`raise`](https://man7.org/linux/man-pages/man3/raise.3.html) function and `kill` command/syscall. There might be other ways, but I'm ignorant about them. The `raise` function is just a process sending a signal to itself. We've looked at the `kill` command, I'll talk about its equivalent function soon. The OS kernel may also send a signal by manipulating the process [struct directly](https://github.com/torvalds/linux/blob/7de25c855b63453826ef678420831f98331d85fd/kernel/signal.c#L1722).

Every signal must have a handler function in a process. This function is executed whenever the process receives the signal. The function can be defined in kernel or user level code. When the OS starts a new process, it assigns default handlers for every signal in that process. Some signals' default handlers just terminate the process. Some others default handlers don't execute anything, i.e the signal is ignored. A signal's default handler can be referenced with the constant `SIG_DFL`. You can see a list of default actions for signals [here](https://elixir.bootlin.com/linux/latest/source/include/linux/signal.h#L352).

A signal default handler can be changed to a different handler. This handler can be a defined function or `SIG_IGN`. `SIG_IGN` tells the process to ignore the signal. We set a signal handler by using either the [`signal()`](https://man7.org/linux/man-pages/man2/signal.2.html) or [`sigaction`](https://man7.org/linux/man-pages/man2/sigaction.2.html) functions. It's also possible that we might want to handle the signal once and then reset the default handler. We can do this by setting the signal's handler function to `SIG_DFL` inside our defined handler.

Enough talk, let's demonstrate a very simple example. Here's a simple python script that execute an infinite loop:

```python
    import signal

    def fpe_bulletproof(signum, frame):
        print("You can't kill me, I'm bullet proof")


    def run():
        signal.signal(signal.SIGFPE, fpe_bulletproof)
        while True: pass

    run()
```

Run this script in a terminal. Open another terminal and look up the pid of the script process using `ps`. Run `kill -8 script_pid` or `kill -fpe script_pid` in the terminal. Now, go to the terminal running the script process, you should see the "You can't kill me, I'm bullet proof" printed in the console. What happened is that we've replaced the default handler with the `fpe_bulletproof` function. Everytime you run the `kill -8 ...` command, the function is executed and the statement is printed to your terminal console. Now try to run `kill -1 script_pid`, you will see that the script has been terminated. This is because we only set a handler for `SIGFPE` and not for others. 

Note that the handler function must have two parameters, the signum and a frame.

Almost all of the signals' handler can be changed, except for `SIGKILL` and `SIGSTOP` signals. These signals cannot be stopped nor ignored. You can try this by changing `SIGFPE` to `SIGKILL` and then rerun the script. An "Invalid Argument" exception is thrown. This is why when you want to force kill a process from the terminal, you type `kill -9 process_pid`. The _9_ represents the `SIGKILL` signal.

A note of warning, you have to be careful of what is executed inside your handler function. Functions called directly and indirectly by your handler function has to be async-safe. Calling async-unsafe functions in your handler function can invoke undefined behaviour. A list of async-safe functions can be found [here](https://man7.org/linux/man-pages/man7/signal-safety.7.html).

### How does this concern IPC?
I know, I know. How does this all concern IPC? Here's how, whenever the `kill pid` is ran in a terminal, a `kill` process is started. This process sends a signal to the process whose pid is included in the command argument. If you squint a little hard, that looks like the `kill` process is communicating with the other process. Isn't that Inter-process communication? Can we have the power that `kill` has?

Luckily for us, yeah we can do what kill does. That's because the `kill` process calls the [`kill()`](https://man7.org/linux/man-pages/man2/kill.2.html) system call. We can send a signal to another process by executing the syscall with its `pid` and the signal. 

Carrying out uni-directional communication from one independent process to another is easy when using `kill()`. All we have to do is run the recipient process, get its pid, and then run the sending process with the pid. Letting both processes know each other pids requires other IPC mechanisms to communicate their pids. What if we don't want to do that? What if we want both processes to kill without knowing each others pids?

We can answer the above questions with two words, process groups. We can start our processes with the same process groups and send signals to each other using `kill(0, signum)`. With this, there's no need for pid exchange and IPC can be carried out in blissful pids ignorance.

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

The above code sets a handler function for `SIGUSR1` signal. This function prints out a statement to the console using `os.write` and increment i. We use `write` rather than `print` because `print` isn't async-safe. It then sends `SIGUSR2` signal in a while loop. When it receives a hundred `SIGUSR1` signals the loop ends and a `SIGQUIT` signal is sent. 

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

This one handles 2 signals, `SIGUSR2` and `SIGQUIT`. Each signal has its own handler function. On receiving a `SIGUSR2` signal, the process prints out a statement and sends a `SIGUSR` signal. When it receives `SIGQUIT` signal, the process sets the boolean variable `should_end` to True. This will end the infinite loop and ensure our program exits.

It is possible to have one handler function for several signals. We can handle each signal differently based on the value of `signum`.

You'd notice that both programs sets the pid parameter of `os.kill` to 0. This works because both processes run in the same process group. Here's the shell script that is used to run the processes

```sh
    trap "" USR1 USR2 QUIT
    python3 server.py & python3 client.py
```

We use the trap instruction to handle the all the signals sent by both Python processes. This is because `kill(0, sig)` sends a signal to all the processes in a process group, and the shell process is in the same process group with its default handler(termination). We don't want that, and that's why we handle them with an empty statement.

### Performance
Signals are plenty fast. [Cloudflare](https://blog.cloudflare.com/scalable-machine-learning-at-cloudflare/#ipc-mechanisms) benchmarked 	404,844 1KB messages per second. That's really cool!

### Demo Code
You can find my code on UDS on [GitHub](https://github.com/goodyduru/ipc-demos).

### Conclusion
Unix signals are a straightforward but limited mechanism for IPC. They can do much more than IPC, e.g. handling errors.  You have to be careful on how you use it though!  

The next article will mechanism that I didn't know existed until recently, it's called Message Queues. Till then, take care of yourself and stay hydrated! ✌🏾