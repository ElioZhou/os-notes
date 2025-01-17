# Processes and Threads

A "processor" can only run one unit of execution (process/thread) at a time. The processor is switched (context switch) among multiple applications so all will appear to be progressing (albeit potentially at reduced speed). The processor and I/O devices can be used efficiently: When application performs I/O, the processor can be used for a different application.

## What is a process?

An abstraction of a running program.

### The Process Model

* A process has a program, input, output, and state (data).
* A process is an instance of an executing program and includes
  * Variables ( memory )
  * Code
  * Program counter ( really hardware resource)
  * Registers
  * ...

### Process: a running program

A process includes:

* Address space
* Process table entries (state, registers): Open files, thread(s) state, resources field

A process tree:

* A created two child processes, B and C
* B created three child processes, D, E and F

```
     A
   /   \
   B    C
 / | \
 D E F
```

### Address Space

* Defines where sections of data and code are located in 32 or 64 address space.
* Defines protection of such sections: ReadOnly, ReadWrite, Execute
* Confined "private" addressing concept: requires form of address virtualization

![Address Space Example](.gitbook/assets/address-space.png)

### Process Creation

* System initialization
  * At boot time
  * Foreground
* Background (daemons)
* Execution of a process creation system call by a running process
* A user request
* A batch job
* Created by OS to provide a service
* Interactive login

### Process Termination

* Normal exit (voluntary)
* Error exit (voluntary)
* Fatal error (involuntary)
* Killed by another process (involuntary)

## Implementation of Processes

* OS maintains a process table `Process procs[];`
* An array (or a hash table) of structures
* One entry per process (pid is the uniq id)

## Implementation of Processes: Process Control Block (PCB)

* Contains the process elements
* It is possible to interrupt a running process and later resume execution as if the interrupt had not occurred → state
* Created and managed by the operating system
* Key tool that allows support for multiple processes

Includes: Identifier, state, priority, program counter, memory pointers, context data, I/O status information and accounting information.

## Fork

Creation of a new process by `fork()`. Executing a program in that new process. Signal notifications.

The kernel boot manually creates ONE process (the init process, pid=0) and all other processes are created by `fork()`.

### `fork()`

```c
#include <stdio.h>
#include <unistd.h>
int main(int argc, char **argv) {
  pid_t pid = fork(); // syscall that creates new PCB and duplicates Address Space
  if (pid == 0) {
    // child process
  } else if (pid > 0) {
    // parent process
  } else {
    // fork failed
    printf("fork() failed!\n");
    return 1;
  }
}
```

fork() will return different values depending on it's whether parent or child. In parent process it returns child pid, and in child process it returns 0.

```cpp
/* fork/getpid test */
#include <sys/types.h>
#include <unistd.h>     /* fork(), getpid() */
#include <stdio.h>

int main(int argc, char* argv[])
{
    int pid;

    printf("Entry point: my pid is %d, parent pid is %d\n",
           getpid(), getppid());

    pid = fork();
    if (pid == 0) {
        printf("Child: my pid is %d, parent pid is %d\n",
               getpid(), getppid());
    }
    else if (pid > 0) {
        printf("Parent: my pid is %d, parent pid is %d, my child pid is %d\n",
               getpid(), getppid(), pid);
    }
    else {
        printf("Parent: oops! can not create a child (my pid is %d)\n",
               getpid());
    }

    return 0;
}
```

Result:

```
Entry point: my pid is 16051, parent pid is 2249
Parent: my pid is 16051, parent pid is 2249, my child pid is 16052
Child: my pid is 16052, parent pid is 16051
```

### `execv()`

The `exec()` family of functions replaces the current process image with a new process image.

## Process State Model: Five-State Model

![Process Five State Model](.gitbook/assets/process-five-state-model.png)

### Using Queues to Manage Processes

![Single blocked queue](.gitbook/assets/using-queues-to-manage-processes.png)

![Multiple blocked queues](.gitbook/assets/using-queues-to-manage-processes-multiple.png)

## Multiprogramming

* One CPU and several processes
* CPU switches from process to process quickly

Running the same program several times will not result in the same execution times due to:

* interrupts
* multi-programming

## Concurrency vs. Parallelism

* Concurrency is when two or more tasks can start, run, and complete in overlapping time periods. It doesn't necessarily mean they'll ever both be running at the same instant. For example, multitasking on a single-core machine.
* Parallelism is when tasks literally run at the same time, e.g., on a multicore processor.

## Threads

* Multiple threads of control within a process: unique execution
* All threads of a process share the same address space and resources (with exception of stack)

### Why Threads?

* For some applications many activities can happen at once:
  * With threads, programming becomes easier
    * Otherwise application needs to actively manage different logical executions in the process
    * This requires significant state management
  * Benefit applications with I/O and processing that can overlap
* Lighter weight than processes
* Can be used to implement concurrency
  * Faster to create and restore: we just really need a stack and an execution unit, but don't have to create new address space etc.

### Processes vs. Threads

* Process groups resources: Address Space, files
* Threads are entities scheduled for execution on CPU
* Threads can be in any of several states: running, blocked, ready, and terminated (remember the process state model?)
* No protections among threads (unlike processes) \[Why?] → this is important
  * _Since every thread can access every memory address within the process’ address space, one thread can read, write, or even completely wipe out another thread’s stack._
  * _There is no protection between threads because (1) it is impossible, and (2) it should not be necessary_
* The unit of dispatching is referred to as a thread or lightweight process (lwp)
* The unit of resource ownership is referred to as a process or task (unfortunately in linux struct task represents both a process and thread)
* Multithreading: The ability of an OS to support multiple, concurrent paths of execution within a single process
* Process is the unit for resource allocation and a unit of protection.
* Process has its own (one) address space.
* A thread has:
  * an execution state (Running, Ready, etc.)
  * saved thread context when not running
  * an execution stack
  * some per-thread static storage for local variables
  * access to the memory and resources of its process (all threads of a process share this)

## Kernel-Level Threads (KLTs)

Thread management is done by the kernel. No thread management is done by the application.

### Advantages:

* The kernel can simultaneously schedule multiple threads from the same process on multiple processors
* If one thread in a process is blocked, the kernel can schedule another thread of the same process
* Kernel routines can be multithreaded

### Disadvantages:

* The transfer of control from one thread to another within the same process requires a mode switch to the kernel

### Implementing Threads in Kernel Space

* Kernel knows about and manages the threads
* No runtime is needed in each process
* Creating/destroying/(other thread related operations) a thread involves a system call

Advantages:

* When a thread blocks (due to page fault or blocking system calls) the OS can execute another thread from the same process

Disadvantages:

* Scalability (operating systems had limited memory dedicated to them), **ULTs scales better.**
* ~~Cost of system call is very high~~ (Disagree because if you want to implement interruption to do thread scheduling you have to use `signal(SIGVTALARM)` which is much more expensive.)

## User-Level Threads (ULTs)

* All thread management is done by the application
* Initially developed to run on kernels that are not multithreading capable
* The kernel is not aware of the existence of threads

### Implementing Threads in User Space

* Threads are implemented by a library
* Kernel knows nothing about threads
* Each process needs its own private thread table in userspace
* Thread table is managed by the runtime system

### Advantages

* Thread switch does not require kernel-mode
* Scheduling (of threads) can be application-specific
* Can run on any OS
* Scales better

### Disadvantages

* A system call by one thread can block all threads of that process
* Page fault blocks the whole process
* In pure ULT, multithreading cannot take advantage of multiprocessing

## PCB vs. TCB

Process Control Block handles global process resources. Thread Control Block handles thread execution resources.

| Per process items           | Per thread items |
| --------------------------- | ---------------- |
| Address space               | Program counter  |
| Global variables            | Registers        |
| Open files                  | Stack            |
| Child processes             | State            |
| Pending alarms              |                  |
| Signals and signal handlers |                  |
| Accounting information      |                  |

## 1:1, M:1, M:N

Thread Models are also known as the general ratio of user threads over kernels threads

* 1:1: each user thread == kernel thread
* M:1: user level thread mode
* M:N: hybrid model

## Context Switch

Scenarios:

* Current process (or thread) blocks _OR_
* Preemption via interrupt

Operations to be done:

* Must release CPU resources (registers)
* Requires storing "all" non provileged registers to the PCB or TCB save area
* Tricky as you need registers to do this
* All written in assembler
* Typically an architecture has a few privileged registers so the kernel can accomplish this

## Conclusions

#### Process/Threads are one the most central concepts in Operating Systems

#### Process vs. Thread (understand differences)

* Process is a resource container with at least one thread of execution
* Process is the unit for resource allocation and a unit of protection.
* Thread is a unit of execution that lives in a process (no thread without a process)
* Threads share the resources of the owning process.

#### Multiprogramming vs multithreading

