# File Systems

## Background

### Shortcomings of Process Address Space

- Virtual address space may not be enough storage for all information
- Information is lost when process terminates, is killed, or computer crashes
- Multiple processes may need the information at the same time
- Information might be obtained from 3rd sources

### Requirements for Long-term Information Storage

- Store very large amount of information
- Information must survive the termination of the process using it
- Multiple processes must be able to access the information concurrently

## Files

Files are the data collections created by users. The file system is one of the
most important parts of the OS to a user

Desirable properties of files:

- Files are stored on disk or other secondary storage and do not disappear when
  a user logs off
- Files have names and can have associated access permissions that permit
  controlled sharing
- Files can be organized into hierarchical or more complex structures to reflex
  the relationships among files

Logical units of information created by processes. Used to model disks instead
of RAM (memory).

Information stored in files must be persistent (i.e. not affected by processes
creation and termination) managed by OS.

The part of the OS that manages files is called the file system.
