---
description: Concurrency & Deadlocks
---

# Concurrency

## Inter-Process Communication (IPC)

Processes often need to work together or at the very least share resources.

### Issues

* Send information
* Mitigate contentions(disagreements) over resources
* Synchronize dependencies

### Race Condition

Race condition is a condition where the behavior(result) of the system depends on exact order of processes running.

e.g. Two processes want to access shared memory at the same time: Read is typically not an issue but write or conditional execution is.

## Intra-Process Communication

Threads often need to work together and access resources (e.g. memory) in the common address space.

### Same Issues

* Send information
* Mitigate contentions(disagreements) over resources
* Synchronize dependencies

Example: Multi-threaded Webserver

* Dispatcher thread deposits work into queue of requests to be processed
* Worker threads will pick work from the queue

## Read-Modify-Write Cycles

Read-Modify-Write cycles are typically an issue. For instance, you read a variable, make a decision and modify the variable.

Examples:

```c
if (var == 0) {
  /* do something */
  var = 1;
}
```

```c
if (var == 0) {
  var = 1;
  /* do something */
}
```

```
ldw r3, @var
cmpwi r3, 0
bne L1
stwi @var, 1
/* some code */
```

These examples will have problems if the same code is executed by two threads concurrently. Consider the race and preemption case with 2 threads.

So, expectation is that code has "consistent view" at data.

## Critical Region / Section

Critical section / region is a protected section in code that accesses a shared resource, which can only executed by at most one process at a time.

### Four conditions to prevent errors

1. No two processes simultaneously in critical region
2. No assumptions made about speeds or numbers of CPUs
3. No process running outside its critical region may block another process
4. No process must wait forever to enter its critical region

## Mutual Exclusion with Busy Waiting

{% embed url="https://b23.tv/gwl9zZ4" %}
【【操作系统】进程间通信—互斥-哔哩哔哩】
{% endembed %}

**Mutual Exclusion**: Only one process (thread) at a time can access any shared variables, memory or resources.

#### Simplest solution: Disable interrupt

Problems:

* Disabling interrupt for user program would be too much privilege
* Won't work in SMP (multi-core systems)

#### Lock variable

If the value of the lock variable is 0, then process sets it to 1 and enters. Other processes have to wait.

Problem: [Read-Modify-Write cycles](./#read-modify-write-cycles)

#### Peterson's Solution

Mathematically correct but not practical, cannot be efficiently implemented.

```cpp
#define FALSE 0
#define TRUE 1
#define N 2 // number of processes

int turn = 0; // whose turn is it?
int interested[N]; // all values initially 0

void enter_region(int process) { // process is 0 or 1
  int other = 1 - process; // the other process
  interested[process] = TRUE; // show that you are interested
  turn = process; // set flag
  while (turn == process && interested[other] == TRUE) {
    /* busy wait */
  }
}

void leave_region(int process) { // process: who is leaving
  interested[process] = FALSE; // indicate departure from critical region
}
```

#### The TSL Instruction (Test and Set Lock)

With a "little help" from hardware: the test and set instruction is used to write(set) 1 to a memory location and return its old value as a single **atomic (non-interruptible)** operation.

This is implemented by locking the bus.

```
enter_region:
  TSL REGISTER, LOCK   | copy lock to register and set lock to 1
  CMP REGISTER, #0     | was lock 0?
  JNE enter_region     | if not, loop
  RET                  | return to caller; critical region entered

leave_region:
  MOVE LOCK, #0        | set lock to 0
  RET                  | return to caller
```

#### The XCHG Instruction (cmp\_and\_swap)

This is also implemented by locking the bus.

```
enter_region:
  MOVE REGISTER, #1    | put a 1 in the register
  XCHG REGISTER, LOCK  | swap the contents of the register and lock variable
  CMP REGISTER, #0     | was lock 0?
  JNE enter_region     | if not, loop
  RET                  | return to caller; critical region entered

leave_region:
  MOVE LOCK, #0        | set lock to 0
  RET                  | return to caller
```

#### Load / Store Conditional

Modern processors resolve cmp\_and\_swap through the cache coherency mechanism.

Reservation (`ldwx`) remembers ONE address on a CPU and verifies on (`stwx`) whether still held otherwise store will fail.

```
L1:
  ldwx r3, @lockvar    // load r3 and set reservation register on CPU with &lockvar
  add r3, r3, #1       // increment r3
  stwx r3, @lockvar    // store r3 back to lockvar only if reservation still held
  bcond L1             // if store conditionally failed, loop
```

Reservation is lost on:

1. interrupts
2. if another CPU steals cacheline holding `lockvar`
3. if another `lwdx` is issued.

#### Comparisons

Both TSL (xchg/cmpswp) and Peterson's solution are correct, but they:

* rely on busy-waiting during "contention"
* waste CPU cycles (one way to circumvent this is by calling thread\_yield to voluntarily give up the CPU and upon rescheduling it will attempt again)
* [Priority Inversion problem](./#priority-inversion-problem)

### Priority Inversion Problem

Higher priority process can be prevented from entering a critical section (CS) because the lock variable is dependent on a lower priority process.

## Lock Contention

**Lock Contention** arises when a process/thread attempts to acquire a lock and the lock is not available.

This is a function of (is related to):

* frequency of attempts to acquire the lock
* lock hold time (time between acquisition and release)
* number of threads/processes acquiring a lock

If lock contention is low, [TSL](./#the-tsl-instruction-test-and-set-lock) is an OK solution. The linux kernel uses it extensively for many locking scenarios.

## Concurrency vs Parallelism

**Concurrency** is having multiple contexts of execution not necessarily running at the exact same time.

**Parallelism** is having multiple contexts of execution running at the exact same time.

##
