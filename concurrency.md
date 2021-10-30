---
description: Concurrency & Deadlocks
---

# Concurrency

## Inter-Process Communication (IPC)

Processes often need to work together or at the very least share resources.

### Issues

- Send information
- Mitigate contentions(disagreements) over resources
- Synchronize dependencies

### Race Condition

Race condition is a condition where the behavior(result) of the system depends
on exact order of processes running.

E.g. Two processes want to access shared memory at the same time: Read is
typically not an issue but write or conditional execution IS.

## Intra-Process Communication

Threads often need to work together and access resources (e.g. memory) in the
common address space.

### Same Issues

- Send information
- Mitigate contentions(disagreements) over resources
- Synchronize dependencies

Example: Multi-threaded Webserver

- Dispatcher thread deposits work into queue of requests to be processed
- Worker threads will pick work from the queue

## Read-Modify-Write Cycles

Read-Modify-Write cycles are typically an issue. For instance, you read a
variable, make a decision and modify the variable.

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

```asm
ldw r3, @var
cmpwi r3, 0
bne L1
stwi @var, 1
/* some code */
```

These examples will have problems if the same code is executed by two threads
concurrently. Consider the race and preemption case with 2 threads.

So, expectation is that code has "consistent view" at data.

## Critical Region / Section

Critical section / region is a protected section that accesses a shared
resource, which can only executed by at most one process at a time.

**Mutual Exclusion**: Only one process (thread) at a time can access any shared
variables, memory or resources.

### Four conditions to prevent errors

1. No two processes simultaneously in critical region
2. No assumptions made about speeds or numbers of CPUs
3. No process running outside its critical region may block another process
4. No process must wait forever to enter its critical region

### Mutual Exclusion with Busy Waiting

#### Simple solution: Disable interrupt

Problems:

- Disabling interrupt for user program would be too much privilege
- Won't work in SMP (multi-core systems)

#### Solution: Use lock variable

If the value of the lock variable is 0, then process sets it to 1 and enters.
Other processes have to wait.

Problem: [Read-Modify-Write cycles](#read-modify-write-cycles)

### Peterson's Solution

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

### The TSL Instruction (Test and Set Lock)

With a "little help" from hardware: the test and set instruction is used to
write(set) 1 to a memory location and return its old value as a single atomic
(non-interruptible) operation.

This is implemented by locking the bus.

```asm
enter_region:
  TSL REGISTER, LOCK   | copy lock to register and set lock to 1
  CMP REGISTER, #0     | was lock 0?
  JNE enter_region     | if not, loop
  RET                  | return to caller; critical region entered

leave_region:
  MOVE LOCK, #0        | set lock to 0
  RET                  | return to caller
```

### The XCHG Instruction (cmp_and_swap)

This is also implemented by locking the bus.

```asm
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

### Load / Store Conditional

Modern processors resolve cmp_and_swap through the cache coherency mechanism.

Reservation (`ldwx`) remembers ONE address on a CPU and verifies on (`stwx`)
whether still held otherwise store will fail.

```asm
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

### Comparisons

Both TSL (xchg/cmpswp) and Peterson's solution are correct, but they:

- rely on busy-waiting during "contention"
- waste CPU cycles (one way to circumvent this is by calling thread_yield to
  voluntarily give up the CPU and upon rescheduling it will attempt again)
- [Priority Inversion problem](#priority-inversion-problem)

### Priority Inversion Problem

Higher priority process can be prevented from entering a critical section (CS)
because the lock variable is dependent on a lower priority process.

## Lock Contention

**Lock Contention** arises when a process/thread attempts to acquire a lock and
the lock is not available.

This is a function of (is related to):

- frequency of attempts to acquire the lock
- lock hold time (time between acquisition and release)
- number of threads/processes acquiring a lock

If lock contention is low, [TSL](#the-tsl-instruction-test-and-set-lock) is an
OK solution. The linux kernel uses it extensively for many locking scenarios.

## Concurrency vs Parallelism

**Concurrency** is having multiple contexts of execution not necessarily running
at the exact same time.

**Parallelism** is having multiple contexts of execution running at the exact
same time.

## Producer-Consumer Problem

### Example: Piping

`ls -ls | grep "yooh" | awk '{print $1}'`

#### Responsibility of a Pipe

- Provide Buffer to store data from stdout of Producer and release it to stdin
  of Consumer
- Block Producer when the buffer is full (because consumer has not consumed
  data)
- Block Consumer if no data in buffer when the consumer wants to read (stdin)
- Unblock Producer when buffer space becomes free
- Unblock Consumer when buffer data becomes available

#### Pipes

Pipes are not just for stdin and stdout. They can be created by applications
used for all kinds of things.

PipeBuffer typically has 16 write slots. 4KB guaranteed to be atomic.

#### Race Condition

```cpp
#define N 100
int count = 0;

void producer()
{
  int item = produce_item();         // generate next item
  if (count == N) sleep();           // if buffer is full, go to sleep
  insert_item(item);                 // put item in buffer
  count = count + 1;                 // increment count of items in buffer
  if (count == 1) wakeup(consumer);  // was buffer empty?
}

void consumer()
{
  if (count == 0) sleep();              // if buffer is empty, go to sleep
  int item = remove_item();             // take item out of buffer
  count = count - 1;                    // decrement count of items in buffer
  if (count == N - 1) wakeup(producer); // was buffer full?
  consume_item(item);                   // consume/print item
}
```

Fatal race condition in this example:

1. Let count = 1
2. Consumer begins loop, decrements count == 0
3. COnsumer returns to loop beginning and executes: `if (count == 0)`, then
   preemption happens
4. Producer gets to run, executes `count = count + 1; if (count == 1)` and calls
   `wakeup(consumer)`
5. Preemption Consumer calls `sleep(consumer)`

#### Requirements

- Need a mechanism that allows synchronization between processes/threads on the
  base of shared resources.
- Synchronization implies interaction with scheduling subsystem.
- Let to the innovation of semaphores.

## Semaphore

### Semaphore Data Structure

```cpp
class Semaphore
{
  int value;                // counter
  Queue<Thread*> waiting;   // queue of threads waiting on semaphore

  void Init(int v);         // initialization
  void P();                 // acquiring the semaphore: down(), wait()
  void V();                 // release the semaphore: up(), signal()
}
```

Implementation:

```cpp
void Semaphore::Init(int v)
{
  value = v;
  waiting = Queue<Thread*>.init();  // initialize empty queue
}

void Semaphore::P()
{
  value = value - 1;
  if (value < 0) {
    waiting.add(current_thread);
    current_thread.status = BLOCKED;
    schedule(); // forces wait, thread blocked
  }
}

void Semaphore::V()
{
  value = value + 1;
  if (value <= 0) {
    Thread *thd = waiting.getNextThread();
    scheduler->add(thd); // make it schedulable
  }
}
```

### How do P and V avoid race condition?

`P()` and `V()` must be **atomic**.

#### Solution: By disabling interruptions

First line of `P()` and `V()` can disable interrupts. Last line of `P()` and
`V()` re-enables interrupts.

However, disabling interrupts only works on **single CPU systems**.

#### Solution: Use atomic lock variable

```cpp
class Semaphore
{
  int lockvar;            // to guarantee atomicity
  int value;              // counter
  Queue<Thread*> waiting; // queue of threads waiting on semaphore

  void Init(int v);       // initialization
  void P();               // acquiring the semaphore: down(), wait()
  void V();               // release the semaphore: up(), signal()
}
```

New implementation:

```cpp
void Semaphore::P()
{
  lock(&lockvar); // +
  value = value - 1;
  if (value < 0) {
    waiting.add(current_thread);
    current_thread.status = BLOCKED;
    unlock(&lockvar); // +
    schedule(); // forces wait, thread blocked
  } else {
    unlock(&lockvar); // +
  }
}

void Semaphore::V()
{
  lock(&lockvar); // +
  value = value + 1;
  if (value <= 0) {
    Thread *thd = waiting.getNextThread();
    scheduler->add(thd); // make it schedulable
  }
  unlock(&lockvar); // +
}
```

### Two kinds of semaphores: Mutex and Counting

**Mutex semaphores** or **binary semaphores** or simply LOCK is for mutual
exclusion problems: value initialized to 1.

**Counting semaphores** is for synchronization problems: value initialized to
any value 0..N. Value shows available tokens to enter or number of processes
waiting when negative.

They are of same implementation, just different initial values.

### Semaphore Solution to the Producer-Consumer Problem

3 semaphores (minimal) example:

```cpp
#define N <somenumber>
Semaphore empty = N;
Semaphore full = 0;
Semaphore mutex = 1;
T buffer[N];
int widx = 0, ridx = 0;

Producer(T item)
{
  P(&empty);
  P(&mutex); // Lock
  buffer[widx] = item;
  widx = (widx + 1) % N;
  V(&mutex); // Unlock
  V(&full);
}

Consumer(T &item)
{
  P(&full);
  P(&mutex); // Lock
  item = buffer[ridx];
  ridx = (ridx + 1) % N;
  V(&mutex); // Unlock
  V(&empty);
}
```

4 semaphores for lower lock contention:

```cpp
#define N <somenumber>
Semaphore empty = N;
Semaphore full = 0;
Semaphore mutex_w = 1;
Semaphore mutex_r = 1;
T buffer[N];
int widx = 0, ridx = 0;

Producer(T item)
{
  P(&empty);
  P(&mutex_w); // Lock
  buffer[widx] = item;
  widx = (widx + 1) % N;
  V(&mutex_w); // Unlock
  V(&full);
}

Consumer(T &item)
{
  P(&full);
  P(&mutex_r); // Lock
  item = buffer[ridx];
  ridx = (ridx + 1) % N;
  V(&mutex_r); // Unlock
  V(&empty);
}
```

Using two mutexes for read and write makes it possible for producer and consumer
to be more concurrent (they should not be competing), thus less lock contention.

## Mutexes in Pthreads

Some of the Pthreads calls relating to mutexes are:

| Thread call           | Description               |
| --------------------- | ------------------------- |
| Pthread_mutex_init    | Create a mutex            |
| Pthread_mutex_destroy | Destroy an existing mutex |
| Pthread_mutex_lock    | Acquire a lock or block   |
| Pthread_mutex_trylock | Acquire a lock or fail    |
| Pthread_mutex_unlock  | Release a lock            |

Some of the Pthreads calls relating to condition variables are:

| Thread call            | Description                                  |
| ---------------------- | -------------------------------------------- |
| Pthread_cond_init      | Create a condition variable                  |
| Pthread_cond_destroy   | Destroy a condition variable                 |
| Pthread_cond_wait      | Block waiting for a signal                   |
| Pthread_cond_signal    | Signal another thread and wake it up         |
| Pthread_cond_broadcast | Signal multiple threads and wake all of them |

P: wait, V: signal

### Using threads to solve the producer-consumer problem

```cpp
#include <stdio.h>
#include <pthread.h>
#define MAX 1000000000          // how many numbers to product
pthread_mutex_t the_mutex;
pthread_cond_t condc, condp;
int buffer = 0;                 // buffer used between producer and consumer

void *producer(void *ptr)       // produce data
{
  int i;
  for (i = 0; i < MAX; i++) {
    pthread_mutex_lock(&the_mutex);  // get exclusive access to buffer
    while (buffer != 0) {
      pthread_cond_wait(&condp, &the_mutex);
    }
    buffer = i;                      // put item in buffer
    pthread_cond_signal(&condc);     // wake up consumer
    pthread_mutex_unlock(&the_mutex); // release access to buffer
  }
  pthread_exit(0);
}

void *consumer(void *ptr)         // consume data
{
  int i;
  for (i = 0; i < MAX; i++) {
    pthread_mutex_lock(&the_mutex);  // get exclusive access to buffer
    while (buffer == 0) {
      pthread_cond_wait(&condc, &the_mutex);
    }
    printf("%d\n", buffer);         // consume item from buffer
    buffer = 0;                      // take item out of buffer
    pthread_cond_signal(&condp);     // wake up producer
    pthread_mutex_unlock(&the_mutex); // release access to buffer
  }
  pthread_exit(0);
}

int main(int argc, char **argv) {
  pthread_t pro, con;
  pthread_mutex_init(&the_mutex, 0);
  pthread_cond_init(&condc, 0);
  pthread_cond_init(&condp, 0);
  pthread_create(&con, 0, consumer, 0);
  pthread_create(&pro, 0, producer, 0);
  pthread_join(pro, 0);
  pthread_join(con, 0);
  pthread_cond_destroy(&condc);
  pthread_cond_destroy(&condp);
  pthread_mutex_destroy(&the_mutex);
}
```

## Problems with Semaphores

- It can be difficult to write semaphores code (arguably)
- One has to be careful with the code construction
- If a thread dies and it holds a semaphore, the implicit token is lost

## Some Advice for Locks

- **Always** acquire multiple locks in the same order
- Preferably release in reverse order as acquired: not required but good hygiene
  (Lots of discussion on this, there are scenarios where that is not desired but
  highly optimized implementation)
- Example: SMP CPU scheduler where load balancing is required.

### Example: SMP Scheduler

#### Deadlock example

```cpp
Schedule(int i)
{
  lock(rqlock[i]);
  {
    // load balance with j
    lock(rqlock[j]);
    // pull some threads from j to i
    unlock(rqlock[j]);
  }
  unlock(rqlock[i]);
}
```

Consider situation: `cpu0 (i=0, j=1)` and `cpu1 (i=1, j=0)`, this can lead to
deadlocks -> must use same order instead

#### Force order solution

We force the correct order. If necessary, we release owned lock first and
re-acquire.

```cpp
Schedule(int i)
{
  lock(rqlock[i]);
  {
    // load balance with j
    add_lock(i, j);
    // pull some threads from j to i
    unlock(rqlock[j]);
  }
  unlock(rqlock[i]);
}

void add_lock(int hlv, int alv) // hlv: holding lock, alv: acquiring lock
{
  if (hlv > alv) { // could be "<" or based on addresses, doesn't matter
    // first unlock hlv
    unlock(rqlock[hlv]);
    lock(rqlock[alv]);
    // then lock hlv
    lock(rqlock[hlv]);
  } else {
    lock(rqlock[alv]);
  }
}
```

## Busy Lock vs Semaphore

If lock hold time is short and/or code is uninterruptible, then lock variables
and busy waiting is OK (linux kernel uses it all the time).

Otherwise, use semaphores.

## Other Unix/Linux Mechanisms

- File based: flock()
- System V semaphores: heavy weight as each call is a system call going into the
  kernel.

  semget(), semop() [P and V]

- Futexes: lighter weight as uncontested cases are resolved done using cmpxchg
  in userspace and if race condition is recognized it goes into kernel.

  futex()

- Message queues

  mq_open(), mq_close(), mq_send(), mq_receive()
