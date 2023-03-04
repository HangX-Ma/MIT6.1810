# Lab10: mmap

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

The _mmap_ and _munmap_ system calls allow UNIX programs to exert detailed control over their address spaces. They can be used to share memory among processes, to map files into process address spaces, and as part of user-level page fault schemes such as the garbage-collection algorithms discussed in lecture. In this lab you'll add _mmap_ and _munmap_ to xv6, focusing on memory-mapped files.

## Your Job(hard)

### _kernel/proc.h_

```c
#define NVMA        (16)

struct vma {
  int valid;      // 0 valid, 1 invalid
  uint64 addr;    // Virtual address at which to map the file
  int length;     // Number of bytes to map, 0 mean invalid map
  int prot;       // Permission: PROT_READ or PROT_WRITE or both
  int flags;      // MAP_SHARED(meaning that modifications to the mapped memory 
                  // should be written back to the file) or MAP_PRIVATE(should not)
  int off;        // offset
  struct file* f; // file descriptor
};

...

// Per-process state
struct proc {
  ...
  struct vma vma[NVMA];        // Virtual memory areas
}
```

### _kernel/sysfile.c_

```c
uint64
sys_mmap(void)
{
  uint64 addr;
  int length, prot, flags, fd, offset, i;
  struct file *f;

  argaddr(0, &addr);
  argint(1, &length);
  argint(2, &prot);
  argint(3, &flags);
  argfd(4, &fd, &f);
  argint(5, &offset);
  
  if (length == 0) {
    return -1;
  }

  // Conflict check
  if ((!f->readable && (prot & PROT_READ)) ||
      ((!f->writable) && (prot & PROT_WRITE) && !(flags & MAP_PRIVATE))) {
    return -1;
  }

  length = PGROUNDUP(length);

  uint64 vaend = MAXVA - 2 * PGSIZE;
  // Find a free vma, and calculate where to map the file along the way.
  // We map from high to low.
  struct proc *p = myproc();
  for (i = 0; i < NVMA; i++) {
    /* Find a free slot for mapping */
    struct vma *vv = &p->vma[i];
    if (vv->valid == 0) {
      break;
    }
    else if(vv->addr < vaend) {
      vaend = PGROUNDDOWN(vv->addr);
    }
  }

  if (i == NVMA) {
    panic("syscall mmap");
  }

  struct vma *v = &p->vma[i];
  v->valid  = 1;
  v->addr   = vaend - length;
  v->length = length;
  v->prot   = prot;
  v->flags  = flags;
  v->f      = f;
  v->off    = offset;
  filedup(v->f);  //increase the ref of the file

  return v->addr;
}

uint64
sys_munmap(void)
{

  uint64 addr;
  int length;

  argaddr(0, &addr);
  argint(1, &length);

  if (length == 0) {
    return -1;
  }

  struct proc *p = myproc();
  struct vma *v = 0;
  for (int i = 0; i < NVMA; i++) {
    struct vma *vv = &p->vma[i];
    if (vv->valid == 1 && addr >= vv->addr && addr < vv->addr + vv->length) {
      v = vv;
    }
  }
  if (v == 0) {
    return -1;
  }

  if(addr > v->addr && addr + length < v->addr + v->length) {
    return -1; // hole
  }

  uint64 addr_aligned = addr;
  if(addr > v->addr) {
    addr_aligned = PGROUNDUP(addr);
  }

  int nunmap = length - (addr_aligned - addr); // nbytes to unmap
  if(nunmap < 0)
    nunmap = 0;

  vmaunmap(p->pagetable, addr_aligned, nunmap, v); // custom memory page unmap routine for mmapped pages.
  if(addr <= v->addr && addr + length > v->addr) { // unmap at the beginning
    v->off += addr + length - v->addr;
    v->addr = addr + length;
  }
  v->length -= length;

  if(v->length <= 0) {
    fileclose(v->f);
    v->valid = 0;
  }

  return 0;
}

int 
vmalazy(uint64 va) {
  struct proc *p = myproc();
  // search the suitable vma slot
  struct vma *v = 0;
  for (int i = 0; i < NVMA; i++) {
    struct vma *vv = &p->vma[i];
    if (vv->valid == 1 && va >= vv->addr && va < vv->addr + vv->length) {
      v = vv;
    }
  }
  if (v == 0)
    return -1;

  // allocate physical page
  void *pa = kalloc();
  if(pa == 0) {
    panic("usertrap(): kalloc");
  }
  memset(pa, 0, PGSIZE);

  begin_op();
  ilock(v->f->ip);
  readi(v->f->ip, 0, (uint64)pa, v->off + PGROUNDDOWN(va - v->addr), PGSIZE); //copy a page of the file from the disk
  iunlock(v->f->ip);
  end_op();

  if(mappages(p->pagetable, va, PGSIZE, (uint64)pa, PTE_U | (v->prot << 1)) < 0) {
    panic("pagefault map error");
  }

  return 0;
}
```

### _kernel/trap.c_

```c
void
usertrap(void)
{
    ...
    else if ((r_scause() == 13 || r_scause() == 15)) {
    // lazy allocation
    uint64 va = r_stval();
    if (vmalazy(va) < 0) {
      goto out;
    }
  }
  else {
out:
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }
  ...
}
```

### _kernel/vm.c_

```c

// Remove n BYTES (not pages) of vma mappings starting from va. va must be
// page-aligned. The mappings NEED NOT exist.
// Also free the physical memory and write back vma data to disk if necessary.
void
vmaunmap(pagetable_t pagetable, uint64 va, uint64 nbytes, struct vma *v)
{
  uint64 a;
  pte_t *pte;

  // printf("unmapping %d bytes from %p\n",nbytes, va);

  // borrowed from "uvmunmap"
  for(a = va; a < va + nbytes; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("sys_munmap: walk");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("sys_munmap: not a leaf");
    if(*pte & PTE_V){
      uint64 pa = PTE2PA(*pte);
      if((*pte & PTE_D) && (v->flags & MAP_SHARED)) { // dirty, need to write back to disk
        begin_op();
        ilock(v->f->ip);
        uint64 aoff = a - v->addr; // offset relative to the start of memory range
        if(aoff < 0) { // if the first page is not a full 4k page
          writei(v->f->ip, 0, pa + (-aoff), v->off, PGSIZE + aoff);
        } else if(aoff + PGSIZE > v->length){  // if the last page is not a full 4k page
          writei(v->f->ip, 0, pa, v->off + aoff, v->length - aoff);
        } else { // full 4k pages
          writei(v->f->ip, 0, pa, v->off + aoff, PGSIZE);
        }
        iunlock(v->f->ip);
        end_op();
      }
      kfree((void*)pa);
      *pte = 0;
    }
  }
}
```

### _kernel/proc.c_

```c
static struct proc*
allocproc(void)
{
  ...
  for(int i = 0;i < NVMA; i++) {
    p->vma[i].valid = 0;
  }

  return p;
}


static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  for(int i = 0; i < NVMA; i++) {
    struct vma *v = &p->vma[i];
    vmaunmap(p->pagetable, v->addr, v->length, v);
  }
  ...
}

int
fork(void)
{
  ...
  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  // copy vmas created by mmap.
  // actual memory page as well as pte will not be copied over.
  for(i = 0; i < NVMA; i++) {
    struct vma *v = &p->vma[i];
    if(v->valid) {
      np->vma[i] = *v;
      filedup(v->f);
    }
  }
  ...
}
```

### _kernel/syscall.h_

```c
...
#define SYS_mmap   22
#define SYS_munmap 23
```

### _kernel/syscall.c_

```c
...
extern uint64 sys_mmap(void);
extern uint64 sys_munmap(void);

static uint64 (*syscalls[])(void) = {
...
[SYS_mmap]    sys_mmap,
[SYS_munmap]  sys_munmap,
}
```

### _kernel/riscv.h_

```c
#define PTE_D (1L << 7) // dirty
```

### _user/user.h_

```c
void *mmap(void *, int, int, int, int, int);
int munmap(void *, int);
```

### _user/usys.pl_

```pl
entry("mmap");
entry("munmap");
```
