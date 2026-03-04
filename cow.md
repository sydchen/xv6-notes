# xv6 Copy-on-Write Fork

https://pdos.csail.mit.edu/6.1810/2025/labs/cow.html

## Table of Contents
1. [What is Copy-on-Write Fork?](#what-is-copy-on-write-fork)
2. [Core Concepts](#core-concepts)
3. [Implementation Architecture](#implementation-architecture)
4. [Pseudo Code](#pseudo-code)
5. [Key Design Decisions](#key-design-decisions)
6. [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)

---

## What is Copy-on-Write Fork?

Traditional `fork()` creates a complete copy of the parent process's memory for the child. This is wasteful because:

- `fork()` is almost always followed by `exec()`, which discards all the copied pages anyway
- Even without `exec()`, many pages may never be written to by the child

**Copy-on-Write (COW)** is an optimization that defers the actual copying until it's truly necessary:

- On `fork()`, parent and child **share the same physical pages**
- Pages are marked **read-only** in both processes
- When either process tries to **write** to a shared page, a **page fault** triggers
- At that point, a private copy is made for the writing process
- Only then is the physical memory actually duplicated

### Intuitive Analogy

Think of it like a shared document in the cloud: everyone reads the same copy. Only when someone starts editing do they get their own separate version. COW applies the same principle to memory pages.

---

## Core Concepts

### 1. PTE_COW Flag

To distinguish COW pages from genuinely read-only pages, we need a way to mark a page as "writable, but currently shared — copy before writing."

RISC-V PTEs have **RSW (Reserved for Software)** bits that the hardware ignores but software can use freely. We define:

```
PTE_COW = bit 8 (one of the RSW bits)
```

A COW page has:
- `PTE_W = 0` (write-protected from hardware's perspective)
- `PTE_COW = 1` (software marker: this page is COW, not truly read-only)

This distinction is crucial: when a write fault occurs, we check `PTE_COW` to know whether to copy (COW fault) or kill the process (genuine read-only violation).

### 2. Reference Counting

When multiple processes share a physical page, we need to know when it's safe to free it. A page can only be freed when **no process** references it anymore.

We maintain a **reference count** for every physical page:

```
page_ref[physical_addr / PGSIZE] = number of PTEs pointing to this page
```

Key operations:
- `kalloc()` → set ref count to 1
- `fork()` sharing a page → increment ref count
- `kfree()` → decrement ref count; only free if count reaches 0

### 3. The COW Page Fault

When a process writes to a COW page:

1. Hardware triggers a write page fault (because `PTE_W` is cleared)
2. The fault handler sees `PTE_COW` is set → this is a COW fault
3. Two cases:
   - **ref count == 1**: Only this process uses the page. Safely restore write permission without copying.
   - **ref count > 1**: Other processes still share this page. Allocate a new page, copy content, update PTE.

### 4. Memory Layout (unchanged from normal fork)

```
+-------------------+  ← MAXVA
| TRAMPOLINE        |
+-------------------+  ← MAXVA - PGSIZE
| TRAPFRAME         |
+-------------------+  ← MAXVA - 2*PGSIZE
|                   |
|   (unused space)  |
|                   |
+-------------------+  ← process.sz
|     HEAP          |
+-------------------+
|     DATA          |   ← COW pages initially shared here
+-------------------+
|     TEXT          |   ← Read-only; already not writable, no COW needed
+-------------------+  ← 0x0
```

Text (code) pages are already read-only, so they never need COW treatment — they're naturally shared.

---

## Implementation Architecture

### System Components

```
User Space
  ↓ fork() / write to shared page
Kernel System Calls
  ↓ uvmcopy() — share pages on fork
Page Table Management (vm.c)
  ↓ page fault
Page Fault Handler (vmfault)
  ↓ reference counting
Physical Memory Allocator (kalloc.c)
```

### Key Data Structure

```
Physical Memory Reference Counting:
  pageref.count[PHYSTOP / PGSIZE]  — one counter per page frame
  pageref.lock                      — spinlock for concurrent access
```

### Main Flows

**fork() flow:**
```
fork()
  → uvmcopy():
      for each parent page:
          if page is writable:
              clear PTE_W, set PTE_COW in parent's PTE
          map same physical page into child
          increment physical page ref count
  → child shares parent's physical pages (no copying!)
```

**Write fault on COW page:**
```
write to shared page
  → hardware write fault
  → vmfault():
      detect PTE_COW is set
      if ref count == 1:
          restore PTE_W, clear PTE_COW (no copy needed)
      else:
          allocate new physical page
          copy content from old page
          update PTE to point to new page (writable, not COW)
          decrement ref count on old page (may free it)
```

**kfree() flow:**
```
kfree(page):
  decrement ref count
  if ref count > 0:
      return (other processes still use this page)
  else:
      actually free the page
```

---

## Pseudo Code

### 1. Reference Counting in the Allocator

```c
struct pageref {
    struct spinlock lock;
    int count[PHYSTOP / PGSIZE];
};

void *kalloc(void) {
    void *page = pop from freelist;
    if (page)
        pageref.count[(uint64)page / PGSIZE] = 1;  // new page starts with ref=1
    return page;
}

void kfree(void *page) {
    acquire(&pageref.lock);
    int idx = (uint64)page / PGSIZE;
    if (pageref.count[idx] > 1) {
        pageref.count[idx] -= 1;
        release(&pageref.lock);
        return;                             // still referenced, don't free
    }
    pageref.count[idx] = 0;
    release(&pageref.lock);
    // actually release the page to freelist
}

void kref_inc(void *page) {
    acquire(&pageref.lock);
    pageref.count[(uint64)page / PGSIZE] += 1;
    release(&pageref.lock);
}
```

### 2. fork() — uvmcopy()

```c
int uvmcopy(pagetable_t parent_pagetable, pagetable_t child_pagetable, uint64 size) {
    for (uint64 va = 0; va < size; va += PGSIZE) {
        pte_t *pte = walk(parent_pagetable, va, 0);
        if (!(*pte & PTE_V))
            continue;                       // lazy-allocated page, skip

        uint64 pa = PTE2PA(*pte);
        uint64 flags = PTE_FLAGS(*pte);

        if (flags & PTE_W) {
            flags = (flags | PTE_COW) & ~PTE_W;  // mark COW, clear write
            *pte = PA2PTE(pa) | flags;             // update parent PTE too!
        }

        if (mappages(child_pagetable, va, PGSIZE, pa, flags) < 0)  // share same physical page
            goto on_error;
        kref_inc((void *)pa);              // now two references
    }
    return 0;

on_error:
    uvmunmap(child_pagetable, 0, va / PGSIZE, 1);  // free all child pages mapped so far
    return -1;
}
```

**Why update the parent's PTE?**
If we only mark the child's PTE as read-only but leave the parent writable, the parent can still write freely — defeating the purpose. Both must be write-protected.

### 3. COW Page Fault Handler

```c
uint64 vmfault(pagetable_t pagetable, uint64 va) {
    va = PGROUNDDOWN(va);

    pte_t *pte = walk(pagetable, va, 0);

    // --- Case 1: COW page fault ---
    if (pte && (*pte & PTE_V) && (*pte & PTE_COW)) {
        uint64 pa = PTE2PA(*pte);
        uint64 flags = PTE_FLAGS(*pte);

        if (kref_get((void *)pa) == 1) {
            // We're the only one left — just restore write permission
            *pte |= PTE_W;
            *pte &= ~PTE_COW;
            return pa;
        }

        // Multiple references — must copy
        void *new_page = kalloc();
        if (new_page == 0)
            return 0;                       // out of memory → kill process

        memmove(new_page, (void *)pa, PGSIZE);  // copy old content

        flags = (flags | PTE_W) & ~PTE_COW;
        *pte = PA2PTE((uint64)new_page) | flags;  // point PTE to new private page

        kfree((void *)pa);                  // release our reference on old page
        return (uint64)new_page;
    }

    // --- Case 2: Lazy allocation ---
    if (pte == 0 || !(*pte & PTE_V)) {
        void *new_page = kalloc();
        if (new_page == 0)
            return 0;
        memset(new_page, 0, PGSIZE);
        if (mappages(pagetable, va, PGSIZE, (uint64)new_page, PTE_W | PTE_R | PTE_U) < 0)
            return 0;
        return (uint64)new_page;
    }

    return 0;
}
```

### 4. copyout() — Kernel Writing to User Space

```c
int copyout(pagetable_t pagetable, uint64 dst_va, char *src, uint64 len) {
    while (len > 0) {
        uint64 va = PGROUNDDOWN(dst_va);
        pte_t *pte = walk(pagetable, va, 0);

        // Kernel writes to COW page must trigger copy too
        if (pte && (*pte & PTE_V) && (*pte & PTE_COW)) {
            uint64 pa = vmfault(pagetable, va);
            if (pa == 0)
                return -1;
        }

        uint64 pa = walkaddr(pagetable, va);
        // ... memmove bytes from src to (void *)(pa + offset) ...
    }
}
```

**Why copyout() needs special handling?**
When the kernel copies data into user space (e.g., during a system call like `read()`), it writes to user memory directly — bypassing the hardware page fault mechanism. So we must manually detect and resolve COW pages in this kernel code path.

### 5. Process Exit

```c
void exit(int status) {
    // ... close file descriptors, etc. ...

    // Free all process memory — kfree handles ref counting automatically
    uvmfree(p->pagetable, p->sz);
    // For each freed page, kfree() decrements ref count
    // Physical page is only released when ref count hits 0
}
```

No special COW handling needed in `exit()` — the reference-counted `kfree()` takes care of everything.

---

## Key Design Decisions

### 1. Using RSW Bits for PTE_COW

**Why not reuse an existing flag?**

We could theoretically repurpose an unused bit, but the RISC-V spec explicitly reserves bits 8–9 for supervisor software. Using them is:
- Standard and safe (hardware ignores them)
- Self-documenting (clear intent)
- Non-conflicting with any current or future hardware features

**Alternative considered:** Track COW state in a per-process data structure (like a bitmap). This adds complexity and requires synchronization — RSW bits are simpler and naturally per-PTE.

### 2. Reference Counting with a Global Array

**Approach: Fixed array indexed by physical page number**

```
pageref.count[PHYSTOP / PGSIZE]
```

| Approach | Complexity | Memory Overhead | Lookup Time |
|----------|------------|-----------------|-------------|
| Global array (used) | Low | Fixed (small) | O(1) |
| Embedded in struct run | Medium | Zero | O(1) |
| Separate allocation | High | Variable | O(1) |

**Why a global array?**
- Simple and predictable
- O(1) access via `pa / PGSIZE`
- Memory footprint is small: `PHYSTOP / PGSIZE` integers ≈ a few KB

**Tradeoff:** A single lock (`pageref.lock`) for the entire array creates contention under high parallelism. Fine-grained locking per page or lock striping would improve scalability — not needed for xv6.

### 3. Optimizing the ref count == 1 Case

When a COW fault occurs and the process is the **only** remaining holder of the page, we can skip copying:

```
if ref_count == 1:
    just restore PTE_W and clear PTE_COW
    → no allocation, no copy, O(1)
```

This happens in real workloads: when a child `exec()`s and replaces its address space, many parent pages end up with ref count back to 1. The next write by the parent is then free.

### 4. Updating Both Parent and Child PTEs in uvmcopy()

A subtle but critical point: during `fork()`, we must clear `PTE_W` in the **parent**'s PTEs, not just the child's.

| Scenario | Parent PTE_W | Child PTE_W | Result |
|----------|-------------|-------------|--------|
| Correct | Cleared | Cleared | Both get fault on write |
| Wrong (only child) | Set | Cleared | Parent can write freely; child's copy becomes stale |

If we forget to update the parent, the parent continues writing to the shared physical page, while the child reads those (possibly unexpected) changes — a data consistency bug.

### 5. Handling Text Pages

Text (code) pages are mapped read-only (`PTE_W = 0, PTE_COW = 0`) before COW is even considered. On a write fault to a text page:

```
pte.valid == 1
pte.PTE_W == 0
pte.PTE_COW == 0   ← not COW!
```

The fault handler should detect this and kill the process (illegal write to read-only memory), rather than attempting a copy. The `PTE_COW` check ensures we distinguish the two cases correctly.

---

## Common Pitfalls and Solutions

### 1. Forgetting to Update the Parent's PTE in uvmcopy()

**Wrong:**
```c
// Only set child's PTE as COW
mappages(child_pt, va, PGSIZE, pa, (flags | PTE_COW) & ~PTE_W);
// Parent's PTE still has PTE_W set!
```

**Correct:**
```c
// Update parent's PTE to be COW too
uint64 cow_flags = (flags | PTE_COW) & ~PTE_W;
*pte = PA2PTE(pa) | cow_flags;              // update parent PTE
mappages(child_pt, va, PGSIZE, pa, cow_flags);  // then map child
```

**Consequence:** Without this, the parent can modify the shared page while the child reads it — silent data corruption.

### 2. Decrementing ref count Before Copying

**Wrong:**
```c
kfree((void *)old_pa);              // ref count drops; old page may be freed!
memmove(new_page, (void *)old_pa, PGSIZE);  // use-after-free!
```

**Correct:**
```c
memmove(new_page, (void *)old_pa, PGSIZE);  // copy first
kfree((void *)old_pa);              // then release old reference
```

**Consequence:** If another process frees the page between your `kfree` and your `copy`, you read garbage — or trigger a panic.

### 3. Not Flushing TLB After Updating Parent's PTE

After modifying the parent's PTE in `uvmcopy()`, the TLB may still cache the old (writable) mapping. The kernel must ensure the TLB is flushed before returning from `fork()`.

In xv6, `sfence.vma` flushes the TLB. This is typically done as part of context switching, but be aware that stale TLB entries can allow writes to bypass the write-protection and skip the COW fault entirely.

### 4. Out-of-Memory During COW Fault

If `kalloc()` fails during a COW page fault, the process cannot proceed. The correct behavior is to kill the process:

```c
void *new_page = kalloc();
if (new_page == 0) {
    // Can't recover — kill the process
    p->killed = 1;
    return -1;
}
```

Do not return 0 and silently continue — the user process would be writing to a page it doesn't own.

### 5. Race Condition on Reference Count

The reference count must be modified atomically. A typical race:

```
Process A reads ref count: 1
Process B reads ref count: 1       ← both think they're alone!
Process A skips copy, sets PTE_W
Process B skips copy, sets PTE_W   ← now two processes share a "writable" page!
```

**Solution:** Hold `pageref.lock` for the entire check-and-update sequence, not just the read.

### 6. Double-Freeing a Shared Page

If two processes both decrement the ref count to 0 simultaneously (race), both may try to free the same page:

```
Process A: count = 1 → 0, free page
Process B: count = 1 → 0, free page  ← double free!
```

**Solution:** The decrement and the free decision must be atomic (under the same lock).

### 7. uvmcopy() Partial Failure Cleanup

If `uvmcopy()` fails halfway through (e.g., `mappages` fails), all pages mapped so far need to be released — but their ref counts were already incremented. The cleanup must decrement ref counts correctly, not just free pages:

```c
on_error:
    // for each page already mapped to child:
    uvmunmap(child_pagetable, 0, mapped_pages, 1);  // calls kfree → decrements ref count
    return -1;
```

