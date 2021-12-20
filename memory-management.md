# Memory Management

## What if there's no memory abstraction?

Processes access physical memory directly, so they need to be relocated in order
to not overlap in physical memory.

Can be done at program load time, but it is a bad idea:

- very slow
- Require extra info from program

## Memory Abstraction

To allow several programs to co-exist in memory we need:

- Protection
- Relocation
- Sharing
- Logical organization
- Physical organization

For that, we have a new abstraction for memory: Address Space

## Address Space

Address Space: set of addresses that a process can use to address memory

- Defines where sections of data and code are located in 32 or 64 address space.
- Defines protection of such sections: ReadOnly, ReadWrite, Execute.
- Confined "private" addressing concept: requires form of address
  virtualization.

![Address Space Example](.gitbook/assets/address-space.png)

### Base and Limit

Map each process address space onto a different part of physical memory.

Two registers: Base and Limit.

- Base: start address of a program in physical memory.
- Limit: length of the program

Only OS can modify Base and Limit.

#### Add and Compare

For every memory access:

- Base is added to the address.
- Result is compared to Limit.

This can be done in hardware. So it doesn't significantly add to latency.

## Swapping

- Programs move in and out of memory
- **Holes** are created
- Holes can be combined → **memory compaction**
- What if a process needs more memory?
  - If a hole is adjacent to the process, it is allocated to it
  - Process has to be moved to a bigger hole
  - Process suspended till enough memory is there

## Managing Free Memory

**Bitmap** and **Linked List** are universal methods used in OS and
applications. Other methods employ Heaps.

Bitmap is slow to find k-consecutive 0s for a new process.

Linked List method consists of allocated and free memory segments. It is more
convenient to use **double-linked** lists.

## Buddy Algorithm

Considers blocks of memory only as 2^N.

Potential for fragmentation (drawback).

If no block of a size is available, it splits higher blocks into smaller blocks.

Easy to implement and fast: `O(log2(MaxBlockSz/MinBlockSz))` e.g. 4K .. 128B =
2^(12-7) = 2^(5 steps)

### Examples

![Allocation at level 0](.gitbook/assets/buddy-algorithm.png)

![Free "X" at level 2 leading to coalescing](.gitbook/assets/buddy-algorithm-2.png)

## What are really the problems?

- Memory requirement unknown for most apps.
- Not enough memory: Having enough memory is not possible with current
  technology. How do you determine “enough”?
- Exploit and enforce one condition: Processor does not execute or access
  anything that is not in the memory (how would that even be possible ?) Enforce
  transparently (user is not involved)

## Memory Management Techniques

Memory management brings processes into main memory for execution by the
processor

- involves virtual memory
- based on paging and segmentation

But we can see that...

- All memory references are **logical addresses** in a process’s **address
  space** that are dynamically translated into physical addresses at run time
- An address space may be broken up into a number of pieces that don’t need to
  be contiguously located in main memory during execution.

So it is not necessary that all of the pieces of an address space be in main
memory during execution. Only the ones that I am “currently” accessing

## Virtual Memory

- Each program has its own **address space**
- This address space is divided into **pages** (e.g 4kB)
- Pages are mapped into physical memory chunks (called **frames**)
- By definition then `sizeof(page) == sizeof(frame)` (the size is determined by
  the hardware manufacturer)

### Secondary Storage

Main memory can act as a cache for the secondary storage (disk)

Advantages:

- illusion of having more physical memory
- program relocation
- protection

### Page Table Entry

![Structure of a Page Table Entry](.gitbook/assets/page-table-entry.png)

**Present bit**: '1' if the values in this entry is valid, otherwise translation
is invalid and an pagefault exception will be raised

**Frame Number**: this is the physical frame that is accessed based on the
translation.

**Protection bits**: 'kernel' + 'w' specifies who and what can be done on this
page if kernel bit is set then only the kernel can translate this page. If user
accesses the page a 'privilege exception' will be raised. If _writeprotect_ bit
is set the page can only be read (load instruction). If attempted write (store
instruction), a write protection exception is raised.

**Reference bit**: every time the page is accessed (load or store), the
reference bit is set.

**Modified bit**: every time the page is written to (store), the modified bit is
set.

**Caching Disabled**: required to access I/O devices, otherwise their content is
in cpu cache

### Multi-Level Page Table / RadixTree / Hierarchical Page Table

To reduce storage overhead in case of large memories.

For sparse address spaces (most processes) only few 2nd tables required

#### 4 Level PageTable for 64-bit arch

OS has to make sure no segment is allocated into high range of address space
(63-48 bits). If bits are set, the MMU will raise an exception (this is really
an OS bug then).

![4 Level PageTable for 64-bit arch](.gitbook/assets/4-level-page-table.png)
