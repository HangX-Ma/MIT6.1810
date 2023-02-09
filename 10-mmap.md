# Lab10: mmap

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

The _mmap_ and _munmap_ system calls allow UNIX programs to exert detailed control over their address spaces. They can be used to share memory among processes, to map files into process address spaces, and as part of user-level page fault schemes such as the garbage-collection algorithms discussed in lecture. In this lab you'll add _mmap_ and _munmap_ to xv6, focusing on memory-mapped files.

## Your Job(hard)
