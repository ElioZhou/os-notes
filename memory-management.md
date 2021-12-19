# Memory Management

## Address Space

Address Space:

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
