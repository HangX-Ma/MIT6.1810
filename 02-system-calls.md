# Lab2: system calls

In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. Remember read Chapter 2 of [book-riscv-rev3](book-riscv-rev3.pdf) and Section 4.3 and 4.4 of Chapter 4 before start.

## 1. Using gdb (easy)

The guidance tells us to run `make qemu-gdb` in folder `xv6-labs-2022` and run `riscv64-unknown-elf-gdb` in the same folder but using another terminal. Therefore, you need to install `riscv64-unknown-elf-gdb` first.

### GDB Installation

```bash
# Install prerequisites
$ sudo apt-get install libncurses5-dev python python-dev texinfo libreadline-dev
# Solved: configure: error: GMP is missing or unusable
$ sudo apt install libgmp-dev
# Get source code of GDB
$ wget https://ftp.gnu.org/gnu/gdb/gdb-12.1.tar.gz
$ tar -xvf gdb-12.1.tar.gz
# Configure
$ cd gdb-12.1 && mkdir build && cd build
$ ../configure --prefix=/usr/local --with-python=/usr/bin/python --target=riscv64-unknown-elf --enable-tui=yes
$ make -j$(nproc)
$ sudo make install # needs to be uninstalled manually
# Test
$ riscv64-unknown-elf-gdb --version

GNU gdb (GDB) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```
