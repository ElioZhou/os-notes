# Scheduling

Whether scheduling is based on processes or threads depends on whether the OS is
multi-threading capable: Given a group of ready processes or threads, which
process/thread to run?

## When to schedule?

- When a process is created
- When a process exits
- When a process blocks
- When an I/O interrupt occurs

## Categories of Scheduling Algorithms

- Interactive: preemption is essential, preemption is a means ofr the OS to take
  away the CPU from a currently running process/thread
- Batch:
  - No user impatiently waiting
  - mostly non-preemptive, or preemptive with long period for each process
- Real-time: deadlines

### Scheduling Algorithms: Goals and Measures

- Turn Around Time (Batch)
- Throughput (e.g. Jobs per second)
- Response Time (Interactive)
- Average wait times (how long waiting in ready queue)

## CPU / IO Burst

**CPU Burst**: a sequence of instructions a process runs without requesting I/O.
Mostly dependent on the program behavior.

**IO "Burst"**: time required to satisfy an IO request while the Process can not
run any code. Mostly dependent on system behavior (how many other IOs, speed of
device, etc.)

## Scheduling Algorithms Goals

All systems

- Fairness: giving each process a fair share of the CPU
- Policy enforcement: seeing that stated policy is carried out
- Balance: keeping all parts of the system busy

Batch systems

- Throughput: maximize jobs per hour
- Turnaround time: minimize time between submission and termination
- CPU utilization: keep the CPU busy all the time

Interactive systems

- Response time: respond to requests quickly
- Proportionality: meet users'expectations

Real-time systems

- Meeting deadlines: avoid losing data
- Predictability: avoid quality degradation in multimedia systems

## Process State Transition

Almost ALL scheduling algorithms can be described by the following process state
transition diagram or a derivative of it (we covered some more sophisticated one
in prior lecture)

![Process State Transition](.gitbook/assets/process-state-transition.png)

## Scheduling Algorithms

### First Come First Serve (FIFO / FCFS)

Non-preemptive (run till I/O or exit). Processes ordered as queue: A new process
is added to the end of the queue. A blocked process that becomes ready added to
the end of the queue

Main disadvantage: Can hurt I/O bound processes or processes with frequent I/O

### Shortest Job First

Non-preemptive. Assumes runtime is known in advance. Is only optimal when all
the jobs are available simultaneously.

### Shortest Remaining Time First/Next (SRTF)

Scheduler always chooses the process whose remaining time is the shortest.
Runtime has to be known in advance. Preemptive or non-preemptive (wait till
block or done)

This typically reduces average turnaround time.

### Round Robin

Each process is assigned a time interval referred to as quantum. After the
quantum, the CPU is given to another process (i.e. CPU is removed from the
process/thread aka preemption).

RR = FIFO + preemption/quantum

Length of the quantum

- If too short, too many context switches will result in lower CPU efficiency
- If too long, will cause poor response to short interactive
- quantum longer than CPU burst is good

### Priority Scheduling

Each process is assigned a priority. Runnable process with the highest priority
is allowed to run. Priorities are assigned statically or dynamically.

Must not allow a process to run forever:

- Can decrease the priority of the currently running process
- Use time quantum for each process

### Multiple Level Queuing (MLQ)

Multiple levels of priority (MLQ) plus each level is run round-robin.

Issue: starvation if higher priorities have ALWAYS something to run

### Multi-Level Feedback Queueing (MLFQ)

Aka priority decay scheduler. If process has to be preempted, moves to
(`dynamic_priority--`).

When it reaches "-1", dynamic priority is reset to (`static_priority-1`): This
creates some issues when high prio is reset before low prio is executing.

When a process is made ready (from blocked): its dynamic priority is reset to
(`static_priority-1`).

What kind of process should be in bottom queue?

- Higher priority for IO-Bound tasks
- Lower priority for CPU-Bound tasks

### Comparisons

| Algorithm                     | Advantages                                                                                                                                                                                                                                                                                                              | Disadvantages                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| First Come First Serve (FCFS) | <ul><li>FCFS algorithm doesn't include any complex logic, it just puts the process requests in a queue and executes it one by one.</li><li>Hence, FCFS is pretty simple and easy to implement.</li><li> Eventually, every process will get a chance to run, so starvation doesn't occur.</li></ul>                      | <ul><li>There is no option for pre-emption of a process. If a process is started, then CPU executes the process until it ends.</li><li>Because there is no pre-emption, if a process executes for a long time, the processes in the back of the queue will have to wait for a long time before they get a chance to be executed.</li></ul>                                    |
| Shortest Job First (SJF)      | <ul><li>Each process is served by the CPU for a fixed time quantum, so all processes are given the same priority.</li><li>Starvation doesn't occur because for each round robin cycle, every process is given a fixed time to execute. No process is left behind.</li></ul>                                             | <ul><li>The throughput in RR largely depends on the choice of the length of the time quantum. If time quantum is longer than needed, it tends to exhibit the same behavior as FCFS.</li><li>If time quantum is shorter than needed, the number of times that CPU switches from one process to another process, increases. This leads to decrease in CPU efficiency.</li></ul> |
| Priority based Scheduling     | <ul><li>The priority of a process can be selected based on memory requirement, time requirement or user preference. For example, a high end game will have better graphics, that means the process which updates the screen in a game will have higher priority so as to achieve better graphics performance.</li></ul> | <ul><li>A second scheduling algorithm is required to schedule the processes which have same priority.</li><li>In preemptive priority scheduling, a higher priority process can execute ahead of an already executing lower priority process. If lower priority process keeps waiting for higher priority processes, starvation occurs.</li></ul>                              |

### Usage of Scheduling Algorithms in Different Situations

#### Situation 1: The incoming processes are short and there is no need for the

processes to execute in a specific order.

In this case, FCFS works best when compared to SJF and RR because the processes
are short which means that no process will wait for a longer time. When each
process is executed one by one, every process will be executed eventually.

#### Situation 2: The processes are a mix of long and short processes and the task

will only be completed if all the processes are executed successfully in a given
time.

Round Robin scheduling works efficiently here because it does not cause
starvation and also gives equal time quantum for each process.

#### Situation 3: The processes are a mix of user based and kernel based processes.

Priority based scheduling works efficiently in this case because generally
kernel based processes have higher priority when compared to user based
processes.

For example, the scheduler itself is a kernel based process, it should run first
so that it can schedule other processes.

## Load Balancing (LB)

Occasionally or when no process is runnable, scheduler[i] looks to steal work
elsewhere.

Each scheduler maintains a load average and history to determine stability of
its load. LB typically done in a hierarchy.

Frequency of neighbor check:

- Level in hierarchy: cost to migrate
- Make "small" changes by pulling work from other cpu
