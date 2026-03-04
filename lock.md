# xv6 Locks

https://pdos.csail.mit.edu/6.1810/2025/labs/lock.html

## Table of Contents
1. [Background: Why Lock Contention Hurts](#background-why-lock-contention-hurts)
2. [Lab Tasks Overview](#lab-tasks-overview)
3. [Task 1: Per-CPU Memory Allocator](#task-1-per-cpu-memory-allocator)
4. [Task 2: Read-Write Spinlock](#task-2-read-write-spinlock)
5. [Code Sketches](#code-sketches)
6. [Key Design Decisions](#key-design-decisions)
7. [Common Pitfalls](#common-pitfalls)

---

## Background: Why Lock Contention Hurts

xv6's original memory allocator uses a single global free list:

```
kmem.lock ──► freelist → page → page → page → ...
```

On a multi-core machine, every `kalloc()` and `kfree()` on **every CPU** must acquire the same lock. CPUs spend time spinning waiting for each other rather than doing real work. This is **lock contention**.

The fix is the classic concurrency pattern: **shard the shared resource** so different CPUs usually operate independently.

---

## Lab Tasks Overview

| Task | Problem | Solution |
|------|---------|----------|
| Memory allocator | One lock, all CPUs contend | Per-CPU free lists + stealing |
| Read-write lock | Spinlock treats all accesses equally | Multiple readers, one writer, writer priority |

Both tasks are independent and can be done in any order.

---

## Task 1: Per-CPU Memory Allocator

### The Idea

Give each CPU its own free list, protected by its own lock:

```
CPU 0: kmem[0].lock ──► freelist → page → page → ...
CPU 1: kmem[1].lock ──► freelist → page → page → ...
CPU 2: kmem[2].lock ──► freelist → page → page → ...
CPU 3: kmem[3].lock ──► freelist → page → page → ...
```

In the normal path, CPU 0 allocates from `kmem[0]` only, without touching other CPUs' locks.

### Stealing

When a CPU's own list is empty, it **steals** a page from another CPU's list:

```
Fast path: try own list → success → done
Slow path: own list empty → iterate other CPUs → grab one page from first non-empty list
```

In steady state this slow path is rare, so occasional cross-CPU locking is acceptable.

### push_off / pop_off

`cpuid()` returns the current CPU number. But the scheduler can migrate a thread to a different CPU between the call to `cpuid()` and the use of its result. To prevent this, interrupts must be disabled for the duration:

```c
push_off();          // disable interrupts (prevent migration)
int c = cpuid();     // safe to use now
// ... use kmem[c] ...
pop_off();           // re-enable interrupts
```

`push_off` / `pop_off` are nestable; internally they maintain a depth counter.

### Initial Distribution

`kinit()` calls `freerange()` once. All freed pages land on **one CPU's list** (whichever CPU runs `kinit()`). The other CPUs start empty and will steal on their first allocation. This is fine — stealing redistributes pages naturally.

---

## Task 2: Read-Write Spinlock

### Motivation

A plain spinlock allows only one holder at a time, even for **read-only** operations that don't modify shared state. A read-write lock relaxes this:

| State | Who can enter |
|-------|--------------|
| Idle | Any reader or one writer |
| Readers holding | More readers (concurrently), no writers |
| Writer holding | Nobody else |

This is valuable when reads are frequent and writes are rare — e.g., a file system cache, routing table, or configuration store.

### Writer Priority

Without special handling, a steady stream of readers can starve writers forever — each new reader arrives before the last finishes, so the reader count never reaches zero.

**Writer priority** fixes this: when a writer is waiting, **new readers block** even if the lock is currently readable. Current readers can finish, but no new ones enter.

```
readers_waiting  = 0    (readers see: can I enter?)
writers_waiting  = 0    (writers announce: I am waiting)
writer_active    = 0/1  (is a writer currently holding the lock?)

Reader enters only if:  writer_active == 0 AND writers_waiting == 0
Writer enters only if:  writer_active == 0 AND readers == 0
```

### State Transitions

```
Idle
  ├─ read_acquire()  ──► readers++           ──► Reading (N readers)
  └─ write_acquire() ──► writers_waiting++
                         wait until readers==0
                         writers_waiting--
                         writer_active=1      ──► Writing

Reading (N readers)
  ├─ another read_acquire() (if writers_waiting==0) ──► readers++
  ├─ read_release()  ──► readers--
  │   └─ if readers==0 ──► Idle (writer can now enter)
  └─ write_acquire() ──► writers_waiting++ (blocks new readers)

Writing
  └─ write_release() ──► writer_active=0 ──► Idle
```

### Internal Lock

The rwlock itself uses an inner spinlock. It is held only while updating counters, not during the read/write critical section.

```c
struct rwspinlock {
    struct spinlock lock;    // protects the fields below
    uint     readers;        // number of active readers
    uint     writers_waiting; // number of writers waiting
    uint     writer_active;  // 1 if a writer holds the lock
};
```

The spin-wait loop releases and re-acquires this inner lock repeatedly (busy-wait). That is reasonable here because critical sections are short.

---

## Code Sketches

### 1. Per-CPU kalloc()

```c
void *kalloc(void) {
    push_off();
    int c = cpuid();

    // Fast path: own list
    acquire(&kmem[c].lock);
    struct run *page = kmem[c].freelist;
    if (page)
        kmem[c].freelist = page->next;
    release(&kmem[c].lock);

    // Slow path: steal from another CPU
    if (!page) {
        for (int i = 0; i < NCPU; i++) {
            if (i == c) continue;
            acquire(&kmem[i].lock);
            page = kmem[i].freelist;
            if (page) {
                kmem[i].freelist = page->next;
                release(&kmem[i].lock);
                break;
            }
            release(&kmem[i].lock);
        }
    }

    pop_off();

    if (page)
        memset(page, 1, PGSIZE);   // fill with junk to catch use-after-free
    return (void *)page;
}
```

### 2. Per-CPU kfree()

```c
void kfree(void *pa) {
    struct run *page = (struct run *)pa;
    memset(page, 1, PGSIZE);   // catch dangling refs

    push_off();
    int c = cpuid();

    acquire(&kmem[c].lock);
    page->next = kmem[c].freelist;
    kmem[c].freelist = page;
    release(&kmem[c].lock);

    pop_off();
}
```

### 3. read_acquire()

```c
void read_acquire(struct rwspinlock *rwlk) {
    acquire(&rwlk->lock);

    // Block if a writer is active or waiting (writer priority)
    while (rwlk->writer_active || rwlk->writers_waiting) {
        release(&rwlk->lock);
        acquire(&rwlk->lock);   // spin
    }

    rwlk->readers++;
    release(&rwlk->lock);
    // Lock is now "held" for reading — no inner lock held
}
```

### 4. read_release()

```c
void read_release(struct rwspinlock *rwlk) {
    acquire(&rwlk->lock);
    rwlk->readers--;
    release(&rwlk->lock);
}
```

### 5. write_acquire()

```c
void write_acquire(struct rwspinlock *rwlk) {
    acquire(&rwlk->lock);

    // Announce waiting — blocks new readers from entering
    rwlk->writers_waiting++;

    // Wait until no readers and no other active writer
    while (rwlk->readers > 0 || rwlk->writer_active) {
        release(&rwlk->lock);
        acquire(&rwlk->lock);   // spin
    }

    rwlk->writers_waiting--;
    rwlk->writer_active = 1;
    release(&rwlk->lock);
    // Lock is now held for writing
}
```

### 6. write_release()

```c
void write_release(struct rwspinlock *rwlk) {
    acquire(&rwlk->lock);
    rwlk->writer_active = 0;
    release(&rwlk->lock);
}
```

---

## Key Design Decisions

### 1. Steal One Page at a Time

When the local list is empty, steal **one page** from another CPU, not the entire list.

| Strategy | Locality | Overhead per steal |
|----------|----------|--------------------|
| Steal one page | Page returns to local if freed soon | O(1) |
| Steal half the list | Better amortization | O(n), holds victim's lock longer |

For xv6, one-at-a-time is simpler and good enough. Production allocators often steal in bulk to amortize cost.

### 2. Lock Name Must Start With "kmem"

`kalloctest` checks lock names using a string prefix match. All per-CPU locks must be named `"kmem"` (or `"kmem0"`, `"kmemX"`, etc.). A different name causes test failures even if the logic is correct.

### 3. Writers_waiting Blocks New Readers, Not Current Ones

Writer priority blocks only **new** readers. Readers already inside continue to completion; forcefully ejecting them would require a much more complex design.

So a writer may still wait for current readers, but should eventually get in since new readers are stopped.

### 4. Inner Lock Released During Spin-Wait

The rwlock's spin-wait loop **releases the inner lock** before re-acquiring it:

```c
while (condition) {
    release(&rwlk->lock);
    acquire(&rwlk->lock);   // re-acquire before re-checking
}
```

If the inner lock is held while spinning, no one else can update state and progress stops. Release/re-acquire is the key pattern here.

### 5. writer_active Is a Flag, Not a Count

Only one writer can hold the lock, so a boolean flag (`0` or `1`) is enough. Adding a guard check on release is still useful:

```c
void write_release(struct rwspinlock *rwlk) {
    if (rwlk->writer_active == 0)
        panic("write_release: not locked");
    // ...
}
```

---

## Common Pitfalls

### 1. Calling cpuid() Without Disabling Interrupts

**Wrong:**
```c
int c = cpuid();           // interrupt could migrate thread here
acquire(&kmem[c].lock);    // now operating on wrong CPU's list!
```

**Correct:**
```c
push_off();
int c = cpuid();           // safe: no migration while interrupts off
acquire(&kmem[c].lock);
// ...
pop_off();
```

Thread migration between `cpuid()` and `acquire()` means you'd lock CPU 0's list but execute on CPU 1 — the locked CPU ID and the actual CPU diverge.

### 2. Holding the Inner Lock During the Spin Loop

**Wrong:**
```c
acquire(&rwlk->lock);
while (rwlk->writer_active) {
    // spin here while holding the lock — other threads can't update writer_active!
}
```

**Correct:**
```c
acquire(&rwlk->lock);
while (rwlk->writer_active) {
    release(&rwlk->lock);   // let others update state
    acquire(&rwlk->lock);   // re-acquire before re-checking
}
```

### 3. Forgetting writers_waiting in read_acquire()

**Wrong:**
```c
while (rwlk->writer_active) {   // only checks if writer currently active
    // ...
}
```

A writer waiting with `writers_waiting > 0` is not yet active. Without this check, new readers keep entering, and the waiting writer starves.

**Correct:**
```c
while (rwlk->writer_active || rwlk->writers_waiting) {
    // ...
}
```

### 4. Not Decrementing writers_waiting on Write Acquire

```c
void write_acquire(struct rwspinlock *rwlk) {
    acquire(&rwlk->lock);
    rwlk->writers_waiting++;    // announce waiting

    while (rwlk->readers > 0 || rwlk->writer_active) {
        release(&rwlk->lock);
        acquire(&rwlk->lock);   // spin
    }

    rwlk->writers_waiting--;    // easy to forget this line
    rwlk->writer_active = 1;
    release(&rwlk->lock);
}
```

If this decrement is missed, `writers_waiting` only grows and readers can be blocked forever.

### 5. Forgetting to Steal from All Other CPUs

```c
// Wrong: only tries CPU 0
acquire(&kmem[0].lock);
// steal from kmem[0]
release(&kmem[0].lock);
```

```c
// Correct: iterate all other CPUs
for (int i = 0; i < NCPU; i++) {
    if (i == c) continue;
    // try kmem[i] ...
}
```

If CPU 0 is also empty, that buggy version returns NULL too early.

### 6. Acquiring Two kmem Locks Simultaneously

```c
// Dangerous: holding kmem[c].lock while acquiring kmem[i].lock
acquire(&kmem[c].lock);
struct run *r = kmem[c].freelist;
// list empty, try to steal
acquire(&kmem[i].lock);   // potential deadlock if two CPUs do this simultaneously
```

**Correct:** release local lock before acquiring victim lock. The implementation above never holds two `kmem` locks at once.

---

## Measuring Improvement

The point of this lab is measurable contention reduction. `kalloctest` reports lock acquisition and wait counts:

```
Before (single list):
  kmem: #acquire=100000, #wait=72000   ← high wait count

After (per-CPU lists):
  kmem0: #acquire=30000, #wait=5
  kmem1: #acquire=28000, #wait=3
  ...                                  ← nearly zero contention
```

Wait counts drop because CPUs usually stay on their own free lists. That is the practical payoff of sharding.
