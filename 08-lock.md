# Lab8: locks

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

- [x] [Memory allocator (moderate)](#1-memory-allocator-moderate)
- [x] [Buffer cache (hard)](#2-buffer-cache-hard)

## [1. Memory allocator (moderate)](#lab8-locks)

In function `kalloc()`, I release the CPU lock and create another loop to steal freelist memory from other CPU. This design can prevent deadlock in multiple CPU system. If you hold CPU-A's kmem.lock and acquire CPU-B's kmem.lock, unfortunately, CPU-B also wants to acquire CPU-A's kmem.lock to steal memory, you will trap into a deadlock.

```c
acquire(CPU-A.lock);
...
for (...) {
    acquire(CPU-B.lock);
    if (r) {
        ...
    }
    ...
}
release(CPU-A.lock);
```

### _kernel/kalloc.c_

```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];

void
kinit()
{
  char buf[6];
  for (int i = 0; i < NCPU; i++) {
    snprintf(buf, sizeof(buf), "kmem%d", i);
    initlock(&kmem[i].lock, buf);
  }
  freerange(end, (void*)PHYSTOP);
}

...

void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  push_off();
  int id = cpuid();
  pop_off();

  r = (struct run*)pa;

  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);
}

void *
kalloc(void)
{
  struct run *r;


  push_off();
  int id = cpuid();
  pop_off();

  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  release(&kmem[id].lock);

  /* No free space in current CPU freelist. I release 
  preceding cpu lock to prevent deadlock. */
  if (!r) {
    for (int i = (id + 1) % NCPU; i != id; i = (i + 1) % NCPU) {
        
      acquire(&kmem[i].lock);
      r = kmem[i].freelist;
      if(r) {
        kmem[i].freelist = r->next;
        release(&kmem[i].lock);
        break;
      }
      release(&kmem[i].lock);
    }
  }

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

## [2. Buffer cache (hard)](#lab8-locks)

I check the previous lab requirements, such as 6.s081 lab and find ,in former years, one hint tells you to use timestamp to record the usage of cache buffer. `ticks` is a global variable maintained by `clockintr()`, which can be used as timestamp. Meanwhile, you will notice that the hint indicates the usage of hash chain. Hash buckets and locks number are identical and necessary prime(recommended 13). Due to the usage of `bucket_lock`, the original `lock` needs to be replaced to suit the data architecture.

In function `bget()`, we need to find the rarely used cache whose reference count has reduced to zero. You may notice that I design a way to iterate the buckets in order, because disorder buckets scan likely leads to deadlock. Although deadlock will happen in current design with certain probabilities, it still reduce risks to minimum.
> Imagine a extreme situation, all buckets exits no suitable cache except `0` and `1`. Bucket 0 acquires a bucket lock and bucket 1 also. They all want to find a free and suitable cache. They will eventually acquire each other's locks.

### _Kernel/bio.c_

```c
#define BNUM 13 // hash table bucket number
#define HASH(x) ((uint)x % BNUM)

struct {
  struct spinlock lock;
  struct spinlock bucket_lock[BNUM];
  struct buf buf[NBUF];
  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head[BNUM];
} bcache;

void
binit(void)
{
  struct buf *b;
  char lockbf[8];
  for (int i = 0; i < BNUM; i++) {
    snprintf(lockbf, sizeof(lockbf), "bcache%d", i);
    initlock(&bcache.bucket_lock[i], lockbf);
    // Create linked list of buffers
    bcache.head[i].prev = &bcache.head[i];
    bcache.head[i].next = &bcache.head[i];
  }
  initlock(&bcache.lock, "bcache");

  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    // bucket 0 is randomly selected
    b->ticks = ticks;
    b->next = bcache.head[0].next;
    b->prev = &bcache.head[0];
    initsleeplock(&b->lock, "buffer");
    bcache.head[0].next->prev = b;
    bcache.head[0].next = b;
  }
}

// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  int bucketID = HASH(blockno);
  acquire(&bcache.bucket_lock[bucketID]);
  // Is the block already cached?
  for(b = bcache.head[bucketID].next; b != &bcache.head[bucketID]; b = b->next){
    if(b->dev == dev && b->blockno == blockno) {
      b->refcnt++; // increase reference
      release(&bcache.bucket_lock[bucketID]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Using timestamp to find the cache rarely used
  // We need to check all buckets
  for (int i = (bucketID + 1) % BNUM; i != bucketID; i = (i + 1) % BNUM) {
    struct buf *rbuf = 0;
    uint rticks = 0xFFFFFFFF;
    acquire(&bcache.bucket_lock[i]);
    for(b = bcache.head[i].prev; b != &bcache.head[i]; b = b->prev){
      if (b->refcnt == 0 && b->ticks < rticks) {
        rticks = b->ticks;
        rbuf = b;
      } // A free and rarely used cache

      if(rbuf != 0) {
        rbuf->dev = dev;
        rbuf->blockno = blockno;
        rbuf->valid = 0;
        rbuf->refcnt = 1;
        rbuf->ticks = ticks; // new cache
        // no one is waiting for it.
        // copy from original brelse() function
        rbuf->next->prev = rbuf->prev;
        rbuf->prev->next = rbuf->next;
        rbuf->next = bcache.head[bucketID].next;
        rbuf->prev = &bcache.head[bucketID];
        bcache.head[bucketID].next->prev = rbuf;
        bcache.head[bucketID].next = rbuf;
        // release in order
        release(&bcache.bucket_lock[i]);
        release(&bcache.bucket_lock[bucketID]);
        acquiresleep(&rbuf->lock);
        return rbuf;
      }
    }
    release(&bcache.bucket_lock[i]);
  }
  panic("bget: no buffers");
}
```

On account of the removal of free-cache process, `brelse()` only needs to update the timestamp when reference count decreases to zero. Cooperate with `bget()`, we have set a rule that timestamp only changed at cache release and allocation stage.

```c
// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  uint bucketID = HASH(b->blockno);
  acquire(&bcache.bucket_lock[bucketID]);
  b->refcnt--;
  if (b->refcnt == 0) {
    // update cache timestamp only when the cache last used
    b->ticks = ticks;
  }
  release(&bcache.bucket_lock[bucketID]);
}
```

For consistency, `bpin()` and `bunpin()` both need switch to bucket_lock.

```c
void
bpin(struct buf *b) {
  uint bucketID = HASH(b->blockno);
  acquire(&bcache.bucket_lock[bucketID]);
  b->refcnt++;
  release(&bcache.bucket_lock[bucketID]);
}

void
bunpin(struct buf *b) {
  uint bucketID = HASH(b->blockno);
  acquire(&bcache.bucket_lock[bucketID]);
  b->refcnt--;
  release(&bcache.bucket_lock[bucketID]);
}
```
