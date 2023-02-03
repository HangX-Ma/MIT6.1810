# Lab6: Multithreading

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

- [x] [Uthread: switching between threads (moderate)](#1-uthread-switching-between-threads-moderate)
- [x] [Using threads (moderate)](#2-using-threads-moderate)
- [x] [Barrier(moderate)](#3-barriermoderate)

This lab will familiarize you with multithreading. You will implement switching between threads in a user-level threads package, use multiple threads to speed up a program, and implement a barrier.

## [1. Uthread: switching between threads (moderate)](#lab6-multithreading)

You can copy _kernel/swtch.S_ file content to _user/uthread_switch.S_ without any change. _kernel/proc.h_ has provide the `struct context`, copy that!

### _user/uthread_switch.S_

```arm
    .text

    /*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

    .globl thread_switch
thread_switch:
    /* YOUR CODE HERE */
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    sd s3, 40(a0)
    sd s4, 48(a0)
    sd s5, 56(a0)
    sd s6, 64(a0)
    sd s7, 72(a0)
    sd s8, 80(a0)
    sd s9, 88(a0)
    sd s10, 96(a0)
    sd s11, 104(a0)

    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)

    ret    /* return to ra */}
```

Function `allocproc()` in _kernel/proc.c_ gives us an example about how to execute the function passed to stack. Remember the stack grows from top to bottom.

- **Q:** _`sthread_switch` needs to save/restore only the callee-save registers. Why?_
- **A:** The caller-save registers are stored on the stack per thread. However, the callee-save registers will get lost after switch! We need to create a space to keep these information unchanged.

```c
struct thread_context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct thread_context context;
};

void 
thread_schedule(void)
{
    ...
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&current_thread->context);
    ...
}

void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)t->stack + STACK_SIZE;
}
```

## [2. Using threads (moderate)](#lab6-multithreading)

- **Q:** _Why are there missing keys with 2 threads, but not with 1 thread?_
- **A:** In _notxv6/ph.c_, the `put_thread()` divide keys into `NKEYS/nthread` number iteration for convenient process. However the `put()` function uses the shared tables. Different threads who process the same key will lead to conflict memory access. These thread may simultaneously `insert()` a new key into the same table at the same position or modify the inner value of the same table. Therefore, the original code leads to incomplete data process.

### _notxv6/ph.c_

```c
static 
void put(int key, int value)
{
  ...
  pthread_mutex_lock(&lock);       // acquire lock
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock);     // release lock
}

int
main(int argc, char *argv[])
{
  ...
  pthread_mutex_init(&lock, NULL); // initialize the lock

  //
  // first the puts
  //
  ...
}
```

## [3. Barrier(moderate)](#lab6-multithreading)

### _notxv6/barrier.c_

```c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread != nthread) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  else {
    /* Only last thread will execute this field code */
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond); // wake up every thread sleeping on cond
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```
