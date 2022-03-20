# Producer-Consumer Problem

## Producer-Consumer Problem

Both synchronize and mutex&#x20;

### Example: Piping

`ls -ls | grep "yooh" | awk '{print $1}'`

#### Responsibility of a Pipe

* Provide Buffer to store data from stdout of Producer and release it to stdin of Consumer
* Block Producer when the buffer is full (because consumer has not consumed data)
* Block Consumer if no data in buffer when the consumer wants to read (stdin)
* Unblock Producer when buffer space becomes free
* Unblock Consumer when buffer data becomes available

#### Pipes

Pipes are not just for stdin and stdout. They can be created by applications used for all kinds of things.

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
3. COnsumer returns to loop beginning and executes: `if (count == 0)`, then preemption happens
4. Producer gets to run, executes `count = count + 1; if (count == 1)` and calls `wakeup(consumer)`
5. Preemption Consumer calls `sleep(consumer)`

#### Requirements

* Need a mechanism that allows synchronization between processes/threads on the base of shared resources.
* Synchronization implies interaction with scheduling subsystem.
* Let to the innovation of semaphores.

## Semaphore

{% embed url="https://b23.tv/UsSyrcV" %}
【操作系统】进程间通信—同步-哔哩哔哩
{% endembed %}

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

First line of `P()` and `V()` can disable interrupts. Last line of `P()` and `V()` re-enables interrupts.

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

**Mutex semaphores** or **binary semaphores** or simply LOCK is for mutual exclusion problems: value initialized to 1.

**Counting semaphores** is for synchronization problems: value initialized to any value 0..N. Value shows available tokens to enter or number of processes waiting when negative.

They are of same implementation, just different initial values.

### Semaphore Solution to the Producer-Consumer Problem

{% embed url="https://b23.tv/pSqYqnZ" %}
【计算机操作系统（期末必考系列）--进程同步生产者消费者（PV）-哔哩哔哩】
{% endembed %}

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

Using two mutexes for read and write makes it possible for producer and consumer to be more concurrent (they should not be competing), thus less lock contention.

## Mutexes in Pthreads

Some of the Pthreads calls relating to mutexes are:

| Thread call             | Description               |
| ----------------------- | ------------------------- |
| Pthread\_mutex\_init    | Create a mutex            |
| Pthread\_mutex\_destroy | Destroy an existing mutex |
| Pthread\_mutex\_lock    | Acquire a lock or block   |
| Pthread\_mutex\_trylock | Acquire a lock or fail    |
| Pthread\_mutex\_unlock  | Release a lock            |

Some of the Pthreads calls relating to condition variables are:

| Thread call              | Description                                  |
| ------------------------ | -------------------------------------------- |
| Pthread\_cond\_init      | Create a condition variable                  |
| Pthread\_cond\_destroy   | Destroy a condition variable                 |
| Pthread\_cond\_wait      | Block waiting for a signal                   |
| Pthread\_cond\_signal    | Signal another thread and wake it up         |
| Pthread\_cond\_broadcast | Signal multiple threads and wake all of them |

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

* It can be difficult to write semaphores code (arguably)
* One has to be careful with the code construction
* If a thread dies and it holds a semaphore, the implicit token is lost

## Some Advice for Locks

* **Always** acquire multiple locks in the same order
* Preferably release in reverse order as acquired: not required but good hygiene (Lots of discussion on this, there are scenarios where that is not desired but highly optimized implementation)
* Example: SMP CPU scheduler where load balancing is required.

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

Consider situation: `cpu0 (i=0, j=1)` and `cpu1 (i=1, j=0)`, this can lead to deadlocks -> must use same order instead

#### Force order solution

We force the correct order. If necessary, we release owned lock first and re-acquire.

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

If lock hold time is short and/or code is uninterruptible, then lock variables and busy waiting is OK (linux kernel uses it all the time).

Otherwise, use semaphores.

## Other Unix/Linux Mechanisms

* File based: flock()
*   System V semaphores: heavy weight as each call is a system call going into the kernel.

    semget(), semop() \[P and V]
*   Futexes: lighter weight as uncontested cases are resolved done using cmpxchg in userspace and if race condition is recognized it goes into kernel.

    futex()
*   Message queues

    mq\_open(), mq\_close(), mq\_send(), mq\_receive()

## Monitors

Hoare and Brinch Hansen proposed a higher-level synchronization primitive: monitor.

* Only ONE thread allowed inside the Monitor
* Compiler achieves Mutual Exclusion
* Monitor is a programming language construct like a class or a for-loop

We still need a way to synchronize on events:

* Condition variables: wait & signal
* Not counters: signaling with no one waiting -> event lost
* Waiting on signal releases the monitor and wakeup reacquires it

### Monitor Example

```
monitor example
  integer i;
  condition c;

  procedure producer();
  (* ... *)
  end;

  procedure consumer();
  (* ... *)
  end;
end monitor;
```

### Producer-Consumer Problem with Monitor

```
monitor ProducerConsumer
  condition full, empty;
  integer count;
  procedure insert(item: integer);
  begin
    if count = N then wait(full);
    insert_item(item);
    count := count + 1;
    if count = 1 then signal(empty);
  end;
  function remove: integer;
  begin
    if count = 0 then wait(empty);
    remove:= remove_item();
    count := count - 1;
    if count = N-1 then signal(full);
  end;
  count := 0;
end monitor;
```

## Message Passing

When there's no shared memory (e.g. on distributed systems), we can send LAN messages instead.

`send(destination, &message);`

`send(destination, &message);`

### Producer-Consumer Problem with Message Passing

```cpp
#define N 100 // number of slots in the buffer

void producer(void)
{
  int item;
  message m; // message buffer

  while(TRUE) {
    item = produce_item(); // generate something to put in buffer
    receive(consumer, &m); // wait for an empty to arrive
    build_message(&m, item); // construct a message to send
    send(consumer, &m); // send item to consumer
  }
}

void consumer(void)
{
  int item, i;
  message m;

  for (i = 0; i < N; i++) send(producer, &m); // send N empty messages
  while(TRUE) {
    receive(producer, &m); // get message containing item
    item = extract_item(&m); // extract item from message
    send(producer, &m); // send back empty reply
    consume_item(item); // consume item
  }
}
```

### Message Passing Issues

* Guard against lost messages (acknowledgement)
* Authentication (guard against imposters)

Addressing:

* To processes
* Via Mailbox (place to buffer messages)
* Send to a full mailbox means block
* Receive from an empty mailbox means block

### What about buffer-less messages?

Send and Receive wait (block) for each other to be ready to talk: rendezvous (meet at an agreed time and place)

## Barriers

A barrier for a group of threads or processes means any thread/process must stop at this point and cannot proceed until all other threads/processes reach this barrier.

Three possible states:

1. Processes approaching a barrier
2. All processes but one blocked at the barrier
3. When the last process arrives at the barrier, all of them are let through
