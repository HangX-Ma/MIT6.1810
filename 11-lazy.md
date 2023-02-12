# Lab11: xv6 lazy page allocation (2020)

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

- [x] [Eliminate allocation from sbrk() (easy)](#1-eliminate-allocation-from-sbrk-easy)
- [x] [Lazy allocation (moderate)](#2-lazy-allocation-moderate)
- [x] [Lazytests and Usertests (moderate)](#3-lazytests-and-usertests-moderate)

Lazy lab appears in 2020, which is an significant feature of memory management. I think it is equally important as COW. Mmap lab also uses this feature. `git clone git://g.csail.mit.edu/xv6-labs-2020` and switch to _lazy_ branch. You'll add this lazy allocation feature to xv6 in this lab.

## [1. Eliminate allocation from sbrk() (easy)](#lab11-xv6-lazy-page-allocation-2020)

- **Q:** Try to guess what the result of this modification will be: what will break?
- **A:** When you call `uvmunmap()`, xv6 will check the `PTE_V` flag. After modification, the `sbrk()` only increase the process size but not allocate actual pages, which results in such a 'store page fault'.

### 1.1 _kernel/sysproc.c_

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  struct proc *p = myproc();

  if(argint(0, &n) < 0)
    return -1;
  
  addr = p->sz; // return the old size
  p->sz += n;   // update process size
  return addr;
}
```

## [2. Lazy allocation (moderate)](#lab11-xv6-lazy-page-allocation-2020)

Just follow the hints and copy code from `uvmalloc()` in _kernel/vm.c_.

### 2.1 _kernel/trap.c_

```c
void
usertrap(void)
{
  ...
    else if(r_scause() == 13 || r_scause() == 15) {
    uint64 va = r_stval();

    // Round the faulting virtual address down to a page boundary
    va = PGROUNDDOWN(va);

    char *mem = kalloc();
    memset(mem, 0, PGSIZE);
    if(mappages(p->pagetable, va, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_U) != 0){
      kfree(mem);
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

out:
  ...
}
```

The `uvmumap()` function may encounter the case that one page has not been allocated, this should not panic the xv6. Lazy allocation will deal with it.

### 2.2 _kernel/vm.c_

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      // panic("uvmunmap: walk");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

## [3. Lazytests and Usertests (moderate)](#lab11-xv6-lazy-page-allocation-2020)

- Handle negative `sbrk()` arguments. Deallocate user pages to bring the process size from oldsz to newsz.

### 3.1 _kernel/sysproc.c_

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  struct proc *p = myproc();

  if(argint(0, &n) < 0)
    return -1;
  
  addr = p->sz; // return the old size
  if (n < 0) {
    uvmdealloc(p->pagetable, p->sz, p->sz + n);
  } // part2: Lazy allocation
  p->sz += n;   // update process size
  return addr;
}
```

### 3.2 _kernel/trap.c_

- Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().
- Handle out-of-memory correctly: if kalloc() fails in the page fault handler, kill the current process.
- Handle faults on the invalid page below the user stack.

```c
void
usertrap(void)
{   
    ...
    else if(r_scause() == 13 || r_scause() == 15) {
    uint64 va = r_stval();

    if (va >= MAXVA   /* Out of userspace */ ||
        va >= p->sz  /* Higher than sbrk() allocated vmemory */ ||
        va < PGROUNDUP(p->trapframe->sp) /* Lower than user stack */) {
      p->killed = 1;
      goto out;
    }
    // Round the faulting virtual address down to a page boundary
    va = PGROUNDDOWN(va);

    char *mem = kalloc();
    if(mem == 0){
      p->killed = 1;
      goto out;
    }
    memset(mem, 0, PGSIZE);
    if(mappages(p->pagetable, va, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_U) != 0){
      kfree(mem);
      p->killed = 1;
      goto out;
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

out:
  ...
}
```

- Handle the parent-to-child memory copy in fork() correctly.

### 3.3 _kernel/vm.c_

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  ...
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      // panic("uvmcopy: pte should exist");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmcopy: page not present");
      continue;
    ...
  }
  ...
}
```

The kernel uses function `walkaddr()` to access user space. When `walkaddr()` meets `pte == 0` or `*pte & PTE_V == 0`, page fault occurs, at which we need to allocate and map a page for it.

### 3.4 _kernel/vm.c_

```c
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0 || (*pte & PTE_V) == 0) {
    struct proc *p = myproc();
    if (va >= p->sz  /* Higher than sbrk() allocated vmemory */ ||
        va < PGROUNDUP(p->trapframe->sp) /* Lower than user stack */) {
        return 0;
    }
    // Round the faulting virtual address down to a page boundary
    va = PGROUNDDOWN(va);
    char *mem = kalloc();
    if(mem == 0){
      return 0;
    }
    memset(mem, 0, PGSIZE);
    if(mappages(pagetable, va, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_U) != 0){
      kfree(mem);
      return 0;
    }
    pte = walk(pagetable, va, 0);
  }
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```
