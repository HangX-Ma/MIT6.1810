# Lab4: traps

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

- [x] [RISC-V assembly (easy)](#1-risc-v-assembly-easy)
- [x] [Backtrace (moderate)](#2-backtrace-moderate)
- [x] [Alarm (hard)](#3-alarm-hard)

This lab explores how system calls are implemented using traps.

## [1. RISC-V assembly (easy)](#lab4-traps)

Compile `fs.img` to get `user/call.asm` and then answer the questions below.

### _Answers_

1. **Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?**

    - Up to eight integer registers, `a0–a7`, and up to eight floating-point registers, `fa0–fa7`, are used for this purpose.
    - `a2` is used to hold 13.

2. **Where is the call to function f in the assembly code for main? Where is the call to g?**

    - None. f and g are inlined due to the compiler optimization.

3. **At what address is the function printf located?**

    - 0x64a

4. **What value is in the register ra just after the jalr to printf in main?**

    ```text
    30: 00000097           auipc    ra,0x0
    34: 61a080e7           jalr     1562(ra) # 64a <printf>
    ```

    - pc + 4 = 0x38

5. **Run the following code.**

    ```c
    unsigned int i = 0x00646c72;
    printf("H%x Wo%s", 57616, &i);
    ```

    - What is the output? (_He110 World_)
    - If the RISC-V were instead big-endian what would you set i to in order to yield the same output? (_0x726c6400_)
    - Would you need to change 57616 to a different value? (_No, 57616 is an constant variable_)

6. **In the following code, what is going to be printed after 'y='? Why does this happen?**

    ```c
    printf("x=%d y=%d", 3);
    ```

    - `y=0`. Because 3 is regard as an argument for variable x, leaving y undefined. For those uninitialized variables, ELF file will move them into `.bss` section, whose values are all 0. In other words, register `a1` determines the printed value according to the asm file, in which stores 0.

## [2. Backtrace (moderate)](#lab4-traps)

Only record the _kernel/printf.c_, other modification can be done easily following the hints.

The hints tell us previous stack frame pointer locates at `fp - 16`, and the return address lives at `fp - 8`. All kernel stack consist of one single page-aligned page, which means we can stash current page position to determine whether we have walk though the whole stack page.

### _kernel/printf.c_

```c
void
backtrace(void)
{
  uint64 fp_addr;
  uint64 pgpos;
  uint64 ret_addr;

  fp_addr = r_fp();
  pgpos = PGROUNDDOWN(fp_addr);

  while (pgpos == PGROUNDDOWN(fp_addr)) {
    ret_addr =  *(uint64 *)(fp_addr - 8);
    printf("%p\n", ret_addr);
    fp_addr = *((uint64 *)(fp_addr - 16));
  }
}
```

### _bttest_

```shell
$ bttest
0x00000000800021ac
0x000000008000201e
0x0000000080001d14
```

## [3. Alarm (hard)](#lab4-traps)

You need to finish the test0 first, which will give you the overall understanding of this assignment. test0 only ask you to create some variables to keep user input and handle the time interrupt. `sys_sigalarm()`, `struct proc` and `usertrap()` should pay attention to.

test1-3 deal with interrupt context protection, re-entrant calls prevention and register a0 protection. You can get clues from the following codes.

The hints guide you to create two syscalls. You need to modify the _user/usys.pl_, _user/user.h_, _kernel/syscall.h_, _kernel/syscall.c_ to fullfil the requirements.

```c
    int sigalarm(int ticks, void (*handler)());
    int sigreturn(void);
```

### 3.1 _kernel/proc.c_

```c
int
allocpid()
{
  ...
  // test 1-3: Allocate a trapframe_bk page.
  if((p->trapframe_bk = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  ...
  /* test 0: alarm syscall initialization */
  p->alarm_handler = 0; // an invalid address
  p->alarm_interval = 0; // stop generating periodic alarm calls
  p->ticks_passed = 0;
  /* test 1-3 */
  p->ftag = 0;

  return p;
}

static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  // test 1-3: free the allocated page
  if(p->trapframe_bk) {
    kfree((void*)p->trapframe_bk);
  }
  ...
}
```

### 3.2 _kernel/proc.h_

```c
// Per-process state
struct proc {
  ...
  /* test 1-3: stash trapframe and prevent function re-entrant */
  struct trapframe *trapframe; // data page for trampoline.S
  struct trapframe *trapframe_bk; // backup trapframe to restore registers
  int ftag;                    // prevent re-entrant calls to handler, enabled when 0 is set
  ...
  /* test 0 */
  int alarm_interval;          // alarm ticks
  int ticks_passed;            // alarm ticks passed since last call
  uint64 alarm_handler;        // alarm callback function address
};
```

### 3.3 _kernel/sysproc.c_

```c
uint64
sys_sigalarm(void)
{
  int ticks;
  uint64 handler_addr;
  struct proc *p = myproc();

  argint(0, &ticks);
  argaddr(1, &handler_addr);

  p->alarm_interval = ticks;
  p->alarm_handler = handler_addr;

  return 0;
}


uint64
sys_sigreturn(void)
{
  struct proc *p = myproc();
  
  // copy back the trapframe and reset variables
  *p->trapframe = *p->trapframe_bk;
  p->ticks_passed = 0;
  p->ftag = 0;

  // syscall will return value via register a0, so return
  // p->trapframe->a0 will keep a0 the same as before.
  return p->trapframe->a0;
}
```

### 3.4 _kernel/trap.c_

```c
void
usertrap(void)
{
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2) {
    if (p->alarm_interval == 0 && p->alarm_handler == 0) {
      ;
    }
    else {
      p->ticks_passed++;
      if (p->ticks_passed == p->alarm_interval) {
        /* test 0 */
        // p->trapframe->epc = p->alarm_handler;
        /* test 1-3 */
        if (p->ftag == 0) {
          *p->trapframe_bk = *p->trapframe;
          p->trapframe->epc = p->alarm_handler;
          p->ftag = 1;
        }
      }
    }
    yield();
  }

  usertrapret();
}
```
