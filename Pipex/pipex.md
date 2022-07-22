# Pipex

### Index:
+ [Input:](#input)
+ [Commands are programs!](#commands-are-programs) 
+ [Why processes? ](#why-processes) 
+ [How to communicate between different processes:](#how-to-communicate-between-different-processes)
+ [Redirecting pipes to the input/output of programs:](#redirecting-pipes-to-the-inputoutput-of-programs)
+ [How to write good error messages: ](#how-to-write-good-error-messages)

In this project we are recreating the [UNIX pipelines](https://www.geeksforgeeks.org/piping-in-unix-or-linux/) from the terminal, which redirect output from one command into the input of the next,
using [processes](https://en.wikipedia.org/wiki/Process_(computing)). And we use [perror()](https://linux.die.net/man/3/perror) for the first time, to create better error messages.
<br>

## Input:
For the mandatory part it expects an input file, two commands and an output file in this format: 
`./pipex file1 "cat -e" "grep something" file2` Make sure that the input file exists or create one. It emulates this behaviour: `< infile cmd1 | cmd2 > outfile` Try it out in the terminal to see exactly how it works normally. 
For the bonus part you can execute it in the same way, but with any number of commands. You can use the here_doc by typing: 
`./pipex here_doc DELIMITER "cat -e" "grep something" file2` emulating: `cmd1 << LIMITER | cmd2 >> outfile`
A here_doc is a multi string comment which is taken from STDIN. It will take input untill the given delimiter is given and it will then be used like an input file. 

![unknown](https://user-images.githubusercontent.com/13866954/179375247-7800f6d0-2a32-4499-8f5b-05ea23167022.png)

# Guide
## Commands are programs!
Commands like `ls` are actually programs! They can be found in the folders contained in the [environment variable](https://wiki.archlinux.org/title/environment_variables): PATH. 
If you type `env` you can see all environment variables. And if you type `env | grep PATH` or `echo $PATH`, you will see everything contained in PATH.
When you execute a command, these folders are searched for the right path for the given command and then executed. Essentially providing you with a shortcut, 
so you don't have to type `bin/ls` or remember in which directory to find all these programs. 
This behaviour is part of what we have to write in this project. We are not allowed to use execvpe() which would have done that for us.
You can utilise the [access()](https://linux.die.net/man/2/access) function for this, which checks for valid paths and the [main argument envp](http://crasseux.com/books/ctutorial/Environment-variables.html), which is an array of strings containing the environment variables.


## Why processes? 
To execute these programs in your own program, you need to use one of the [exec functions](https://linuxhint.com/exec_linux_system_call_c/), in this case we are only allowed to use execve. 
If you execute a program like this, it will take over your process and any code written afterwards will be irrelevant, this is why we need
to create child processes which can execute these programs in isolation. The fork function can create a child process. A child process will contain 
a copy of your code and attributes like file descriptors. 
Everything written after that point will be executed by both processes if not further specified which should execute what part of your code. 
And the order of execution is unreliable and will also change in behaviour on different systems. 
This is all controllable however! You can use the return of [fork()](https://linux.die.net/man/2/fork), which will be 0 if you are in the child process. 
And you can use [waitpid()](https://linux.die.net/man/2/waitpid) to wait for a child to finish executing before continuing.

## How to communicate between different processes:
Processes occupy different spaces of memory, so they exist in isolation from one another. So this is where the actual [pipe()](https://www.geeksforgeeks.org/pipe-system-call/) comes in! It is one type of [inter-process communication](https://en.wikipedia.org/wiki/Inter-process_communication) which facilitates communication between processes. It takes in an array of two integers, which it will use to open two filedescriptors that are used for the read and write end of the pipe. The first element, fd[0] will be the read end and fd[1] will be the write end of the pipe. If something is written in the write end, it can be accessed from the other. Create the pipe before creating the child process you want to communicate with, so it gets inherited by the child.
I highly recommend [this](https://www.youtube.com/watch?v=cex9XrZCU14&list=PLfqABt5AS4FkW5mOn2Tn9ZZLLDwA3kZUY) playlist from Codevault about processes.

## Redirecting pipes to the input/output of programs:
The default way for a program to accept input/output is by using the STDIN and STDOUT stream. Every process by default opens these filedescriptors: 
0 - STDIN, 1 - STDOUT, 2 - STDERR. Whenever you use `ls` for example and see the output on your terminal, that is STDOUT. 
So with the dup function you can duplicate a file descriptor. [Dup()](https://linux.die.net/man/2/dup2) will duplicate the given file and give it the value of the first unused file descriptor , which... Is quite useless in this case. With Dup2() however you can do the same thing but specify the value of the new file descriptor, which can be one that is already in use, which is exactly what we want. If you give dup2(fd[1], STDIN_FILENO); This is what happens: 
```
0 - STDIN   -> This stream closes and gets replaced with: 0 - fd[0]
1 - STDOUT                                                1 - STDOUT
2 - STDERR                                                2 - STDERR
3 - fd[0]                                                 3 - fd[0]
4 - fd[1]                                                 4 - fd[1]
```
So now if a program searches for input, it will take the input from the pipe!

## How to write good error messages: 
Most functions from the standard library use Errno (from <errno.h>) which is an integer that can be set to certain macros which correspond with certain error messages. If you use [perror()](https://www.tutorialspoint.com/c_standard_library/c_function_perror.htm) it will print the last error message to STDERR set to errno by a function where an error occured and you can also add a custom string to be displayed beforehand. It uses STDERR because streams like STDOUT might be occupied/closed by the usage of dup for instance! It is important to check if functions return a certain value on failure and to make sure your program exits, frees things properly and displays a helpful error message, so it's easier to debug and maintain your code. 
