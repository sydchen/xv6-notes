# xv6 Threads

https://pdos.csail.mit.edu/6.1810/2023/labs/thread.html

## Table of Contents
1. [Background: What is a Thread?](#background-what-is-a-thread)
2. [Lab Tasks Overview](#lab-tasks-overview)
3. [Task 1: User-Level Thread Switching (uthread)](#task-1-user-level-thread-switching-uthread)
4. [Task 2: Hash Table with Locks](#task-2-hash-table-with-locks)
5. [Task 3: Barrier with Condition Variables](#task-3-barrier-with-condition-variables)
6. [Code Sketches](#code-sketches)
7. [Design Decisions](#design-decisions)
8. [Common Pitfalls](#common-pitfalls)

---

## Background: What is a Thread?

A **thread** is an independent execution context sharing memory with other threads in the same process. Each thread needs its own stack (local variables, function call frames) and register state (PC, SP, and all saved registers).

Switching from thread A to thread B means:
1. Save A's registers somewhere
2. Restore B's registers
3. Jump to B's saved PC (via `ret` using `ra`)

Everything else (heap, globals, file descriptors) is shared automatically.

### Cooperative vs Preemptive

| Model | When does switch happen? | Requires |
|-------|--------------------------|---------|
| **Cooperative** | Thread voluntarily calls `yield()` | Nothing special |
| **Preemptive** | Timer interrupt forces switch | Kernel support |

In this lab, uthread uses **cooperative** scheduling in user space, without kernel preemption support.

---

## Lab Tasks Overview

| Task | Where | What you implement |
|------|-------|--------------------|
| uthread | xv6 user space | `thread_create()`, `thread_schedule()`, `thread_switch.S` |
| Hash table | Linux/macOS POSIX | Add mutex locks to eliminate data races |
| Barrier | Linux/macOS POSIX | Condition variable barrier with round tracking |

Tasks 2 and 3 run on POSIX pthreads (Linux/macOS), not inside xv6.

---

## Task 1: User-Level Thread Switching (uthread)

### Data Structures

```c
struct context {
    uint64 ra;          // return address — where to resume
    uint64 sp;          // stack pointer
    uint64 s0 ... s11;  // callee-saved registers (s0–s11)
};

struct thread {
    char           stack[STACK_SIZE];  // 8192 bytes, embedded in struct
    int            state;              // FREE / RUNNABLE / RUNNING
    struct context context;            // saved register state
};
```

Only **callee-saved registers** (`ra`, `sp`, `s0`–`s11`) are saved. Caller-saved registers (`a0`–`a7`, `t0`–`t6`) are the thread's own responsibility by the C calling convention — they are not preserved across a `thread_yield()` call anyway.

### thread_create()

Setting up a new thread so that `thread_schedule()` can run it for the first time:

```c
void thread_create(void (*func)()) {
    // Find a FREE slot
    struct thread *t = find_free_thread();
    t->state = RUNNABLE;

    // Stack pointer: top of the embedded stack (stack grows downward)
    t->context.sp = (uint64)(t->stack + STACK_SIZE);

    // ra: when thread_switch restores ra and executes ret, it jumps here
    t->context.ra = (uint64)func;

    // All other registers start at 0
}
```

Key point: `thread_switch` ends with `ret`, and `ret` jumps to `ra`. So `ra = func` makes the first schedule of that thread enter `func()`.

### thread_schedule()

```c
void thread_schedule(void) {
    // Find next RUNNABLE thread (round-robin)
    struct thread *next = find_next_runnable();

    if (current_thread != next) {
        next->state = RUNNING;
        struct thread *old = current_thread;
        current_thread = next;
        // Save old context, restore new context, jump to next->context.ra
        thread_switch((uint64)&old->context, (uint64)&next->context);
    }
}
```

### thread_switch.S — The Assembly Core

`thread_switch(a0, a1)`:
- `a0` = pointer to old thread's `struct context` (save here)
- `a1` = pointer to new thread's `struct context` (restore from here)

```asm
thread_switch:
    # Save old thread's callee-saved registers into *a0
    sd ra,  0(a0)
    sd sp,  8(a0)
    sd s0,  16(a0)
    ...
    sd s11, 104(a0)

    # Restore new thread's registers from *a1
    ld ra,  0(a1)
    ld sp,  8(a1)
    ld s0,  16(a1)
    ...
    ld s11, 104(a1)

    ret   # jumps to restored ra — the new thread's saved PC
```

After `ret`, execution continues on the new thread's stack at its saved `ra`. From that thread's view, `thread_switch()` simply returned.

### Thread Lifecycle

```
thread_create(func)
  → state = RUNNABLE
  → context.ra = func, context.sp = top of stack

thread_schedule() picks this thread
  → thread_switch() restores context
  → ret → jumps to func()

func() runs, calls thread_yield()
  → state = RUNNABLE
  → thread_schedule() → switches away

func() finishes
  → state = FREE
  → thread_schedule() → never picked again
```

---

## Task 2: Hash Table with Locks

### The Race Condition

The hash table uses separate chaining (linked lists per bucket). A `put(key, value)` operation:
1. Computes `bucket = key % NBUCKET`
2. Walks the bucket's list looking for an existing key
3. If not found, inserts a new node at the head

With two threads doing step 3 simultaneously on the **same bucket**:

```
Thread 1: new_node->next = bucket[b]   // sees old head
Thread 2: new_node->next = bucket[b]   // also sees old head
Thread 1: bucket[b] = new_node1        // installs node1
Thread 2: bucket[b] = new_node2        // overwrites! node1 is lost
```

One `put()` is silently lost — the key goes missing.

### Coarse vs Fine-Grained Locking

**Coarse (one global lock):** correct but slow — only one thread can `put`/`get` at a time.

```c
pthread_mutex_t global_lock;

void put(int key, int value) {
    pthread_mutex_lock(&global_lock);
    // ... insert ...
    pthread_mutex_unlock(&global_lock);
}
```

**Fine-grained (one lock per bucket):** correct and fast — threads on different buckets run in parallel.

```c
pthread_mutex_t locks[NBUCKET];

void put(int key, int value) {
    int b = key % NBUCKET;
    pthread_mutex_lock(&locks[b]);
    // ... insert into bucket b ...
    pthread_mutex_unlock(&locks[b]);
}
```

With per-bucket locks, operations on different buckets can proceed in parallel.

---

## Task 3: Barrier with Condition Variables

### What a Barrier Does

A barrier ensures all N threads reach a synchronization point before any of them continues:

```
Thread 1 ───────────────────►|
Thread 2 ──────►|            |  (waits here)
Thread 3 ──────────────────►|
                              ▼ all continue
```

### Implementation

```c
struct barrier {
    pthread_mutex_t    lock;
    pthread_cond_t     cond;
    int                count;   // threads currently waiting
    int                round;   // which barrier round we're in
    int                nthread; // total threads
};

void barrier() {
    pthread_mutex_lock(&b.lock);

    b.count++;
    if (b.count == b.nthread) {
        // Last thread to arrive — wake everyone
        b.round++;
        b.count = 0;
        pthread_cond_broadcast(&b.cond);
    } else {
        // Wait, but guard against spurious wakeups and the wrap-around issue
        int my_round = b.round;
        while (b.round == my_round)
            pthread_cond_wait(&b.cond, &b.lock);
    }

    pthread_mutex_unlock(&b.lock);
}
```

### Why Track `round`?

Without `round`, a fast thread can re-enter the barrier too early and mix generations, causing stale state and incorrect blocking.

`round` gives each thread a generation number, so it can wait for the right broadcast:

```
my_round = b.round    // snapshot current round before sleeping
while (b.round == my_round)   // still in the same round → keep waiting
    pthread_cond_wait(...)
// b.round changed → the broadcast we needed has fired
```

---

## Code Sketches

### 1. struct context and struct thread

```c
struct context {
    uint64 ra;    // return address
    uint64 sp;    // stack pointer
    uint64 s0;  uint64 s1;  uint64 s2;  uint64 s3;
    uint64 s4;  uint64 s5;  uint64 s6;  uint64 s7;
    uint64 s8;  uint64 s9;  uint64 s10; uint64 s11;
};

struct thread {
    char           stack[STACK_SIZE]; // STACK_SIZE = 8192
    int            state;
    struct context context;
};
```

### 2. thread_create()

```c
void thread_create(void (*func)()) {
    struct thread *t;
    for (t = all_thread; t < all_thread + MAX_THREAD; t++)
        if (t->state == FREE) break;

    t->state = RUNNABLE;
    memset(&t->context, 0, sizeof(t->context));

    t->context.sp = (uint64)(t->stack + STACK_SIZE); // top of stack
    t->context.ra = (uint64)func;                     // first PC on ret
}
```

### 3. thread_switch.S

```asm
.globl thread_switch
thread_switch:
    sd ra,  0(a0);  sd sp,  8(a0)   # save old ra, sp
    sd s0, 16(a0);  sd s1, 24(a0)   # save old s0-s1
    # ... s2 through s11 at offsets 32–104 ...
    sd s11, 104(a0)

    ld ra,  0(a1);  ld sp,  8(a1)   # restore new ra, sp
    ld s0, 16(a1);  ld s1, 24(a1)   # restore new s0-s1
    # ... s2 through s11 ...
    ld s11, 104(a1)

    ret                               # jump to restored ra
```

### 4. Fine-grained hash table locking

```c
pthread_mutex_t locks[NBUCKET];

// init
for (int i = 0; i < NBUCKET; i++)
    pthread_mutex_init(&locks[i], NULL);

// put
void put(int key, int value) {
    int b = key % NBUCKET;
    pthread_mutex_lock(&locks[b]);
    // insert or update in bucket b
    pthread_mutex_unlock(&locks[b]);
}

// get — also needs lock if put runs concurrently
int get(int key) {
    int b = key % NBUCKET;
    pthread_mutex_lock(&locks[b]);
    // walk bucket b
    pthread_mutex_unlock(&locks[b]);
}
```

### 5. Barrier

```c
void barrier(void) {
    pthread_mutex_lock(&b.lock);

    b.count++;
    if (b.count == b.nthread) {
        b.round++;
        b.count = 0;
        pthread_cond_broadcast(&b.cond);
    } else {
        int my_round = b.round;
        while (b.round == my_round)
            pthread_cond_wait(&b.cond, &b.lock);
    }

    pthread_mutex_unlock(&b.lock);
}
```

---

## Design Decisions

### 1. Save Only Callee-Saved Registers

The C calling convention guarantees that caller-saved registers (`a0`–`a7`, `t0`–`t6`) are **not** preserved across function calls. Any code that calls `thread_yield()` already assumes those registers may be clobbered. So the context switch only needs to preserve what the convention promises to preserve: `ra`, `sp`, `s0`–`s11`.

Saving all 32 registers would still work, but costs extra time and space.

### 2. ra as the First PC

When a new thread runs for the first time, there is no "saved state" to restore — only what `thread_create` set up. Setting `context.ra = func` exploits the fact that `thread_switch` ends with `ret`, which jumps to `ra`. This is equivalent to making `thread_switch` appear to have been called from `func`.

Setting `context.sp` to the top of the embedded stack ensures `func` starts with a valid stack.

### 3. Stack Embedded in struct thread

The stack is embedded in `struct thread` (fixed-size array). This keeps allocation simple: creating a thread just finds a free slot in `all_thread[]`.

Downside: no guard page, so stack overflow can corrupt nearby thread structs. For this teaching lab that tradeoff is acceptable.

### 4. Per-Bucket Locks vs Global Lock

| Locking | Correctness | Performance |
|---------|------------|-------------|
| None | Broken (lost keys) | Fast |
| One global lock | Correct | Poor (no parallelism) |
| One lock per bucket | Correct | Good (parallel on different buckets) |

The benchmark target (>=1.25x with 2 threads) effectively requires per-bucket locking.

### 5. round Counter in Barrier

Condition variables are stateless — a `broadcast` that fires before a `wait` is lost. Without `round`, threads that loop back to the barrier early may sleep through the next broadcast or see a stale count.

`round` acts as a generation counter. Each thread snapshots `my_round` and waits until `b.round != my_round`, so wakeup matches the correct barrier round.

---

## Common Pitfalls

### 1. Setting sp to the Bottom Instead of Top of Stack

Stacks on RISC-V grow downward. The stack pointer must start at the top (highest address) of the allocated region.

Starting at the bottom:
```c
t->context.sp = (uint64)t->stack;           // bottom — first push goes below!
```

Start at the top:
```c
t->context.sp = (uint64)(t->stack + STACK_SIZE);  // correct
```

### 2. Setting ra to 0 or Leaving It Unset

If `ra` is 0 when `thread_switch` executes `ret`, the new thread jumps to address 0 — instant segfault.

`thread_create` must set `context.ra = (uint64)func` before the thread is ever scheduled.

### 3. Saving Only Some Registers in thread_switch.S

If any callee-saved register is missing from the save/restore sequence, the thread that gets switched away will find a corrupted register when it resumes.

The offsets must exactly match the layout of `struct context`:
```
ra  →  0
sp  →  8
s0  → 16
s1  → 24
...
s11 → 104
```

### 4. Race on Non-Bucket-Local Data in Hash Table

If `put` also modifies a global counter or global list, locking per-bucket is insufficient — the shared global still races.

Keep locking scope small, but include **all** shared mutable state.

### 5. Using `if` Instead of `while` for Condition Variable Wait

`pthread_cond_wait` can have spurious wakeups — it can return even when no `broadcast` was called. An `if` check doesn't re-test the condition:
```c
if (b.round == my_round)
    pthread_cond_wait(&b.cond, &b.lock);
```

Always loop:
```c
while (b.round == my_round)
    pthread_cond_wait(&b.cond, &b.lock);
```

### 6. Barrier: Resetting count After broadcast, Not Before

If `count` is reset after `broadcast`, a fast thread can re-enter and contaminate the next round.

Reset `count = 0` **before** `broadcast` (while the lock is still held), so no thread can re-enter the barrier until after `round` has incremented.

```c
b.round++;
b.count = 0;               // reset before broadcast
pthread_cond_broadcast(&b.cond);
```

### 7. Not Updating current_thread Before thread_switch

`thread_switch` changes the stack pointer. If `current_thread` still points to the old thread when `thread_switch` returns (on the new stack), any code that uses `current_thread` after the switch operates on the wrong thread.

```c
// Correct order:
struct thread *old = current_thread;
current_thread = next;          // update BEFORE switch
thread_switch(&old->context, &next->context);
// now running on next's stack; current_thread is correct
```

---

## How Uthread Relates to the Kernel Scheduler

The uthread design closely mirrors xv6 kernel `swtch()`:

| | uthread | xv6 kernel |
|--|---------|------------|
| Context struct | `struct context` in user space | `struct context` in `kernel/proc.h` |
| Switch function | `thread_switch()` in user assembly | `swtch()` in `kernel/swtch.S` |
| Scheduling | `thread_schedule()` — round-robin | `scheduler()` — round-robin |
| Registers saved | `ra`, `sp`, `s0`–`s11` | same |
| Stack | embedded char array | `p->kstack` (kernel page) |
| Preemption | No (cooperative only) | Yes (timer interrupt → `yield()`) |

The key difference: kernel threads are preempted by timer interrupts, while uthreads only switch when a thread explicitly calls `thread_yield()`.
