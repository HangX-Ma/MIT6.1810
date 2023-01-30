# Lab3: page tables

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

- [x] [Speed up system calls (easy)](#1-speed-up-system-calls-easy)
- [ ] [Print a page table (easy)](#2-print-a-page-table-easy)
- [ ] [Detect which pages have been accessed (hard)](#3-detect-which-pages-have-been-accessed-hard)

In this lab you will explore page tables and modify them to speed up certain system calls and to detect which pages have been accessed.

## [1. Speed up system calls (easy)](#lab3-page-tables)

Speed up certain system calls by sharing data in a read-only region between userspace and the kernel. `USYSCALL` is determined as the shared region address.

### 1.1 _kernel/proc.h_

Hint3 tells us to allocate and initialize page in `allocproc()`. We know `getpid()` is a syscall that obtain _pid_ from `struct proc` so we can use this structure as the shared pid container.

```c
// Per-process state
struct proc {
  ...
  struct trapframe *trapframe; // data page for trampoline.S
  struct usyscall *usc;        // User process ID
  struct context context;      // swtch() here to run process
  ...
}
```

### 1.2 _kernel/proc.c_

Allocate memory for user defined variable and load with `p->pid`. This part is forced to finish before `proc_pagetable()` function because we need to do memory mapping in `proc_pagetable()`.

```c
static struct proc*
allocproc(void)
{
  ...
  /*  allocate one page for p->usc */
  if ((p->usc = (struct usyscall *)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  p->usc->pid = p->pid;

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
}
```

```c
pagetable_t
proc_pagetable(struct proc *p)
{
  /* USYSCALL mapping here */
  if(mappages(pagetable, USYSCALL, PGSIZE, 
              (uint64)(p->usc), PTE_R | PTE_U) < 0){
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
}
```

I forget `PTE_U` in preceding code initially, which occurs an error like this.

```shell
== Test   pgtbltest: ugetpid == 
  pgtbltest: ugetpid: FAIL 
    ...
         $ pgtbltest
         ugetpid_test starting
         usertrap(): unexpected scause 0x000000000000000d pid=4
                     sepc=0x000000000000049a stval=0x0000003fffffd000
         $ qemu-system-riscv64: terminating on signal 15 from pid 30173 (make)
    MISSING '^ugetpid_test: OK$'
```

Finally you need to do some cleanup tasks.

```c
// Free a process's page table, and free the
// physical memory it refers to.
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);

  uvmfree(pagetable, sz);
}

static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->usc)
    kfree((void*)p->usc);
  p->usc = 0;
  ...
}
```

Please run `./grade-lab-pgtbl ugetpid` rather than `./grade-lab-pgtbl pgtbltest` to test your code. Otherwise, you may waste an hour and get an error seemingly implies some details haven't been improved.

```shell
== Test   pgtbltest: ugetpid == 
  pgtbltest: ugetpid: OK 
== Test   pgtbltest: pgaccess == 
  pgtbltest: pgaccess: FAIL 
    ...
```

> Don't ask me why I know this. (_**pgtbltest**_ is an alias of the third lab :cry:)

### _Question 1_

Which other xv6 system call(s) could be made faster using this shared page? Explain how.

### _Answer 1_

Those syscalls serve for status information can be made faster. `struct proc` can be expanded with user space variables, such as `uint64 usz`, who uses the shared memory to record the size of process memory. We can modify the `growproc()` to store this variable with direct-memory access.

## [2. Print a page table (easy)](#lab3-page-tables)

### _kernel/vm.c_

```c
void
vmprint_helper(pagetable_t pagetable, int depth)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    if ((pte & PTE_V)) {
      for (int n = 1; n <= depth; n++) {
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      // this PTE points to a lower-level page table.
      if ((pte & (PTE_R | PTE_W | PTE_X)) == 0) {
        uint64 child = PTE2PA(pte);
        vmprint_helper((pagetable_t)child, depth + 1);
      }
    }
  }
}

void
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmprint_helper(pagetable, 1);
}
```

### _Question 2_

Explain the output of vmprint in terms of Fig 3-4 from the text.

- What does page 0 contain? What is in page 2?
- When running in user mode, could the process read/write the memory mapped by page 1?
- What does the third to last page contain?

### _Answer 2_

```shell
page table 0x0000000087f6b000
 ..0: pte 0x0000000021fd9c01 pa 0x0000000087f67000
 .. ..0: pte 0x0000000021fd9801 pa 0x0000000087f66000
 .. .. ..0: pte 0x0000000021fda01b pa 0x0000000087f68000
 .. .. ..1: pte 0x0000000021fd9417 pa 0x0000000087f65000
 .. .. ..2: pte 0x0000000021fd9007 pa 0x0000000087f64000
 .. .. ..3: pte 0x0000000021fd8c17 pa 0x0000000087f63000
 ..255: pte 0x0000000021fda801 pa 0x0000000087f6a000
 .. ..511: pte 0x0000000021fda401 pa 0x0000000087f69000
 .. .. ..509: pte 0x0000000021fdcc13 pa 0x0000000087f73000
 .. .. ..510: pte 0x0000000021fdd007 pa 0x0000000087f74000
 .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
init: starting sh
```

- page 0 contains the `data` and `text` contents, because line 65 and line 68 in `kernel/exec.c` firstly allocate and map memory for ELF file segments. Afterwards, the line 81-88 create two pages, stack and guard page. According to the figure 3.4 in text book Page 2 is the user stack.
- Page 1 locates at the guard page region, with `PTE_U` cleared. Therefore, user cannot read/write this memory.
- I think the third to last page ought to be the remaining part of the memory.

## [3. Detect which pages have been accessed (hard)](#lab3-page-tables)
