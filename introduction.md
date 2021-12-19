---
description: Introduction to Operating Systems
---

# Introduction

Operating systems turn ugly hardware into beautiful abstractions (arguable).

## The Operating System as a Resource Manager

- Top down view: Provide abstractions to application programs
- Bottom up view: Manage pieces of complex systems (hardware and events)
- Alternative view: Provide orderly, controlled allocation of resources

## Two Main Tasks of OS

- Provide programmers (and programs) a clean set of abstract resources and
  services to manipulate these resources
- Manage the hardware resources

## Resources and Services

Resources: Allocation, Protection, Reclamation and Virtualization

Services: Abstraction, Simplification, Convenience and Standardization

### Operating System Short Explanation

OS (kernel) is really just a program that runs with special privileges to
implement the features of allocation, protection, reclamation and virtualization
and the services that are structured on top of it.

## Booting Sequence

- BIOS starts: checks how much RAM, keyboard, other basic devices
- BIOS determines boot Device
- The first sector in boot device is read into memory and executed to determine
  active partition
- Secondary boot loader is loaded from that partition
- This loaders loads the OS from the active partition and starts it.

## OS Services

- Program development
- Program execution
- Access I/O devices
- Controlled access to files
- System access
- Error detection and response
- Accounting

## Operating System Jungle / Zoo

- Mainframe operating systems
- Server operating systems
- Multiprocessor operating systems
- Personal computer operating systems
- Real-time operating systems
- Embedded operating systems
- Smart card operating systems
- Cellphone/tablet operating systems
- Sensor operating systems

## Processors

Each CPU has a specific set of instructions, ISA (Instruction Set Architecture)
largely epitomized in the assembler

- RISC: Sparc, MIPS, PowerPC
- CISC: x86, zSeries

All CPUs contain:

- **General registers**: inside to hold key variables and temporary results
- **Special registers**: visible to the programmer
  - Program counter contains the memory address of the next instruction to be
    fetched
  - Stack pointer points to the top of the current stack in memory
  - PSW (Program Status Word) contains the condition code bits which are set by
    comparison instructions, the CPU priority, the mode (user or kernel) and
    various other control bits

### How Processors Work

Execute instructions in CPU cycles.

- Fetch(from mem) → decode → execute
- Program counter (PC)
- Pipeline: fetch n+2 while decode n+1 while execute n

### CPU Caches

Principle:

- Data/Instruction that were recently used are “likely” used again in short
  period
- Caching is principle used in “many” subsystems ( I/O, filesystems, … ) [
  hardware and software]

Cache hit: no need to access memory

Cache miss: data obtained from mem, possibly update cache

Issues:

- Operation MUST be correct
- Cache management for Memory done in hardware
- Data can be in read state in multiple caches but only in one cache when in
  write state

## OS Major Components

- Process and thread management
- Resource management
  - CPU
  - Memory
  - Device (I/O)
- File system
- Bootstrapping

## Process: a running program

A process includes:

- Address space
- Process table entries (state, registers): Open files, thread(s) state,
  resources held

A process tree:

- A created two child processes, B and C
- B created three child processes, D, E and F

```
     A
   /   \
   B    C
 / | \
 D E F
```

## Address Space

- Defines where sections of data and code are located in 32 or 64 address space.
- Defines protection of such sections: ReadOnly, ReadWrite, Execute
- Confined "private" addressing concept: requires form of address virtualization
