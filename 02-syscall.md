# Lab2: system calls

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

- [x] [Using gdb (easy)](#1-using-gdb-easy)
- [x] [System call tracing (moderate)](#2-system-call-tracing-moderate)
- [x] [Sysinfo (moderate)](#3-sysinfo-moderate)

In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. Remember read Chapter 2 of [book-riscv-rev3](book-riscv-rev3.pdf) and Section 4.3 and 4.4 of Chapter 4 before start.

## [1. Using gdb (easy)](#lab2-system-calls)

The guidance tells us to run `make qemu-gdb` in folder `xv6-labs-2022` and run `riscv64-unknown-elf-gdb` or `gdb-multiarch` in the same folder but using another terminal. Therefore, you need to install `riscv64-unknown-elf-gdb` first.

> Windows WSL 1, Ubuntu20.4 encounters errors using gdb. The  `gdb-multiarch` has passed usage test under Ubuntu20.04, physical machine.

### GDB Installation (Optional)

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
```

```shell
$ riscv64-unknown-elf-gdb --version

GNU gdb (GDB) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

### GDB Test

If you solved the problem following the guidance and hints, you can get the following output on terminal. Run `target remote localhost:26000` first if gdb runs correctly.

```shell
$ gdb-mulitarch
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04.1) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
The target architecture is assumed to be riscv:rv64
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000001000 in ?? ()
(gdb)
```

### Answers

1. **Looking at the backtrace output, which function called syscall?**

    ```shell
    (gdb) backtrace
    #0  syscall () at kernel/syscall.c:133
    #1  0x0000000080001d14 in usertrap () at kernel/trap.c:67
    #2  0x0505050505050505 in ?? ()
    ```

    `backtrace` will display the call stack for the currently selected thread. Stack follows the FILO rules so we can confirm that function `usertrap()` at `kernel/trap.c:67` called syscall.

2. **What is the value of `p->trapframe->a7` and what does that value represent? (Hint: look user/initcode.S, the first user program xv6 starts.)**

    ```shell
    $4 = {lock = {locked = 0x0, name = 0x80008178, cpu = 0x0}, state = 0x4,
    chan = 0x0, killed = 0x0, xstate = 0x0, pid = 0x1, parent = 0x0,
    kstack = 0x3fffffd000, sz = 0x4000, pagetable = 0x87f6c000,
    trapframe = 0x87f74000, context = {ra = 0x80001466, sp = 0x3fffffda60,
    s0 = 0x3fffffda90, s1 = 0x80008d30, s2 = 0x80008900, s3 = 0x0,
    s4 = 0x1030, s5 = 0x10, s6 = 0x2, s7 = 0x87f6c000, s8 = 0x10,
    s9 = 0x1000, s10 = 0x1000, s11 = 0x2000}, ofile = {
    0x0 <repeats 16 times>}, cwd = 0x80016e40, name = {0x69, 0x6e, 0x69,
    0x74, 0x0, 0x0, 0x64, 0x65, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}}
    ```

    The value is **`7`**. We know `trapframe = 0x87f74000` from preceding context. `kernel/syscall.h` has defined `SYS_exec`, whose value is equal to 7. `initnode.S` load immediate `SYS_exec` number into register `a7`, which will be used to call the syscall `sys_exec`.

    - `p /x p->trapframe->a7` can print out the required value straightly.
    - If you look into the architecture of `struct trapframe`, `a7` is the $22^{th}$ element in this container. `x/22ug 0x87f74000` can show the details starting from address `0x87f74000`, totally 22 eight-bytes.
  
    ```shell
    0x87f74000:     9223372036855332863     274877898752
    0x87f74010:     2147490950      24
    0x87f74020:     1       361700864190383365
    0x87f74030:     4096    361700864190383365
    0x87f74040:     361700864190383365      361700864190383365
    0x87f74050:     361700864190383365      361700864190383365
    0x87f74060:     361700864190383365      361700864190383365
    0x87f74070:     36      43
    0x87f74080:     361700864190383365      361700864190383365
    0x87f74090:     361700864190383365      361700864190383365
    0x87f740a0:     361700864190383365      7
    ```

3. **What was the previous mode that the CPU was in?**

    Check the reference book about RISC-V privileged instructions, at page 63.

    _The SPP bit indicates the privilege level at which a hart was executing before entering supervisor
    mode. When a trap is taken, SPP is set to 0 if the trap originated from user mode, or 1 otherwise._

    ```shell
    (gdb) p /x $sstatus
    $2 = 0x22
    ```

    SPP is the eighth bit in register _status_. So `0x22` indicates SPP is actually 0, demonstrating the fact that CPU previous mode is _user_ mode.

4. **Write down the assembly instruction the kernel is panicing at. Which register corresponds to the varialable `num`?**

    ```shell
    scause 0x000000000000000d
    sepc=0x0000000080001ff4 stval=0x0000000000000000
    panic: kerneltrap
    QEMU: Terminated
    ```

    - Instruction `kerneltrap`.
    - `num = * (int *) 0;` is translated to `lw a3,0(zero) # 0 <_entry-0x80000000>` in `kernel/kernel.asm`, where the kernel is panicing at. register `a3` corresponds to the variable `num`.

5. **Why does the kernel crash? Hint: look at figure 3-3 in the text; is address 0 mapped in the kernel address space? Is that confirmed by the value in `scause` above? (See description of `scause` in [RISC-V privileged instructions](https://pdos.csail.mit.edu/6.828/2022/labs/n//github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf))**

    - `lw` read a word from specific address and write it into the `rd(register destination)`. Visual address `0` is not an effective address so the kernel cannot translate.
    - Actually not, Figure 3.3 in [text book](book-riscv-rev3.pdf) has explained the _KERNELBASE_ start at 0x80000000.
    - The exception code is 13 **(Load page fault)**, which means the xv6 kernel cannot read the given address content. This `scause` value confirms our inference.

6. **What is the name of the binary that was running when the kernel paniced? What is its process id (pid)?**

    - `p p->name` prints `"initcode\000\000\000\000\000\000\000"`.
    - `p p->pid` prints `1`.

## [2. System call tracing (moderate)](#lab2-system-calls)

This assignment needs to add a system call tracing feature. The hints will guide you complete a majority of tasks.

### Notice

- An array of syscall names need to follow the definition order in `kernel/syscall.h`.
- The 'mask' provided as an argument for `trace` syscall is actually regarded as a group of buttons. `1` means tracing, otherwise not.

### 2.1 _kernel/syscall.h_

```c
...
#define SYS_trace  22
```

### 2.2 _kernel/syscall.c

Add all syscall names appearing in `kernel/syscall.h` and add `sys_trace` elements. Modify the `syscall` function to print out essential information.

```c
...
extern uint64 sys_trace(void);

// An array mapping syscall numbers from syscall.h
// to the function that handles the system call.
static uint64 (*syscalls[])(void) = {
    ...
    [SYS_trace]   sys_trace,
}

static char* syscall_name[22] = {
  "fork", "exit", "wait", "pipe", "read",
  "kill", "exec", "fstat", "chdir", "dup",
  "getpid", "sbrk", "sleep", "uptime", "open",
  "write", "mknod", "unlink", "link", "mkdir",
  "close", "trace",
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
    /* Check whether the specific syscall is traced or not */
    int maskbit = (p->traced_mask & (1 << num)) >> num;
    if (maskbit) {
      printf("%d: syscall %s -> %d\n", 
              p->pid, syscall_name[num - 1], p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

### 2.3 _kernel/sysproc.c_

```c
// store the traced syscall mask
uint64
sys_trace(void)
{
  int n;
  struct proc *p = myproc();

  argint(0, &n);
  p->traced_mask = n;

  return 0;
}
```

### 2.4 _kernel/proc.h_

```c
struct proc {
  ...
  // these are private to the process, so p->lock need not be held.
  int traced_mask;             // syscall trace argument container 
  ...
};
```

### 2.5 _kernel/proc.c_

```c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  ...
  np->sz = p->sz;
  // copy traced mask from parent to child
  np->traced_mask = p->traced_mask;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);
  ...
}
```

### 2.6 _user/user.h_

```c
// system calls
...
int uptime(void);
int trace(int);
```

### 2.7 _usys.pl_

```c
...
entry("uptime");
entry("trace");
```

## [3. Sysinfo (moderate)](#lab2-system-calls)

The fundamental modifications applied for new syscall is similar to preceding assignment. One exception is that `get_freemem()` and `get_nproc()` declarations needs to be written into _kernel/defs.h_.

### 3.1 _kernel/kalloc.c_

`get_freemem()` refers to `kalloc()`, utilizing the memory `freelist` to calculate the remaining pages. So don't forget to multiply the PAGE_SIZE with the count and get the total free space bytes.

```c
// get free memory page count
uint64
get_freemem(void)
{
  uint64 freemem_count;
  struct run *r;

  freemem_count = 0;
  acquire(&kmem.lock);
  r = kmem.freelist;
  while(r) {
    r = r->next;
    freemem_count++;
  }
  release(&kmem.lock);

  return freemem_count * 4096;
}
```

### 3.2 _kernel/proc.c_

`get_nproc()` refers to `wakeup()`.

```c
// get UNUSED process number
int
get_nproc(void)
{
  int nproc;
  struct proc *p;

  nproc = 0;
  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state != UNUSED) {
      nproc++;
    }
    release(&p->lock);
  }

  return nproc;
}
```

### 3.3 _kernel/sysproc.c_

You need to look into the `sys_fstat()` (_kernel/sysfile.c_) and `filestat()` (_kernel/file.c_). Remember the arguments passing in assembly through register `a0-a7` in RISC-V.

```c
uint64
sys_sysinfo(void)
{
  struct sysinfo sysinfo;
  struct proc *p = myproc();
  uint64 addr; // user pointer to struct sysinfo

  argaddr(0, &addr);

  sysinfo.freemem = get_freemem();
  sysinfo.nproc = get_nproc();

  if(copyout(p->pagetable, addr, (char *)&sysinfo, sizeof(sysinfo)) < 0) {
    return -1;
  }
  return 0;
}
```
