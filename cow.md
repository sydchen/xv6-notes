# xv6 Copy-on-Write Fork

https://pdos.csail.mit.edu/6.1810/2025/labs/cow.html

## Table of Contents
1. [What is Copy-on-Write Fork?](#what-is-copy-on-write-fork)
2. [Core Concepts](#core-concepts)
3. [Implementation Architecture](#implementation-architecture)
4. [Pseudo Code](#pseudo-code)
5. [Design Decisions](#design-decisions)
6. [Common Pitfalls](#common-pitfalls)

---

## What is Copy-on-Write Fork?

Traditional `fork()` creates a full copy of the parent's memory for the child. This is wasteful for two reasons: `fork()` is almost always followed by `exec()`, which throws away all those copied pages immediately; and even without `exec()`, most pages the child inherits will never be written to.

COW defers the actual copying until a write happens. On `fork()`, parent and child share the same physical pages, both mapped read-only. When either tries to write to a shared page, the hardware fires a write fault. The fault handler then allocates a private copy — and only then does real duplication happen.

Think of it like a shared Google Doc: everyone reads the same version until someone starts editing, at which point they branch off. COW does the same for memory pages.

---

## Core Concepts

### 1. PTE_COW Flag

We need to distinguish COW pages from genuinely read-only pages (like text). The difference matters: a write to a COW page should trigger a copy; a write to a read-only text page should kill the process.

RISC-V PTEs have RSW (Reserved for Software) bits — bits 8–9 that hardware ignores completely. We use bit 8 as `PTE_COW`.

A COW page has `PTE_W = 0` (write-protected at hardware level) and `PTE_COW = 1` (our software marker). When a write fault occurs, checking `PTE_COW` tells us which case we're in.

### 2. Reference Counting

When multiple processes share a physical page, we can't free it until everyone is done with it. We track this with a reference count per physical page:

```
page_ref[physical_addr / PGSIZE] = number of PTEs pointing to this page
```

- `kalloc()` initializes the count to 1
- `fork()` sharing a page increments it
- `kfree()` decrements it and only frees when it hits 0

### 3. The COW Page Fault

When a process writes to a COW page:

1. Hardware triggers a write fault (`PTE_W` is cleared)
2. The fault handler sees `PTE_COW` is set → COW fault
3. Two cases:
   - ref count == 1: we're the only one left. Just restore `PTE_W` and clear `PTE_COW` — no copy needed.
   - ref count > 1: others still share this page. Allocate a fresh page, copy the content, update the PTE.

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

Text pages are already read-only, so they're naturally shared without any COW machinery.

---

## Implementation Architecture

### Components

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

### Data Structure

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

Notice we update the parent's PTE, not just the child's. If we left the parent's `PTE_W` set, the parent could keep writing to the shared page while the child reads it — silent data corruption.

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

`copyout()` needs this because when the kernel writes into user space (e.g., during `read()`), it does so directly — bypassing the hardware page fault mechanism entirely. Without this check, the kernel would write through the COW mapping without triggering a copy.

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

No special COW handling needed in `exit()` — reference-counted `kfree()` takes care of everything.

---

## Design Decisions

### 1. RSW Bits for PTE_COW

We could track COW state in a per-process bitmap, but that adds complexity and requires its own synchronization. Using the RISC-V RSW bits (8–9) is simpler: the hardware ignores them, they live right in the PTE where you need them, and using the reserved bits is exactly what they're there for.

### 2. Global Array for Reference Counts

```
pageref.count[PHYSTOP / PGSIZE]
```

| Approach | Complexity | Memory Overhead | Lookup Time |
|----------|------------|-----------------|-------------|
| Global array (used) | Low | Fixed (small) | O(1) |
| Embedded in struct run | Medium | Zero | O(1) |
| Separate allocation | High | Variable | O(1) |

A fixed array indexed by `pa / PGSIZE` is O(1), predictable, and takes only a few KB (`PHYSTOP / PGSIZE` integers). The tradeoff is a single global lock — fine for xv6, but a bottleneck under real parallelism. Lock striping per page range would help there.

### 3. Skipping the Copy When ref count == 1

When a COW fault fires and the process is the only holder of the page, there's nothing to protect anymore — just restore `PTE_W` and clear `PTE_COW`. No allocation, no copy, O(1).

This matters in practice: when a child `exec()`s and replaces its address space, the parent's pages drop back to ref count 1. The parent's next write to those pages is then essentially free.

### 4. Updating Both Parent and Child PTEs in uvmcopy()

During `fork()`, we must clear `PTE_W` in the parent's PTEs too, not only the child's.

| Scenario | Parent PTE_W | Child PTE_W | Result |
|----------|-------------|-------------|--------|
| Correct | Cleared | Cleared | Both get fault on write |
| Wrong (only child) | Set | Cleared | Parent writes freely; child's copy becomes stale |

If the parent's PTE stays writable, the parent can silently modify the shared physical page while the child reads it.

### 5. Handling Text Pages

Text pages are mapped `PTE_W = 0, PTE_COW = 0` before any of this runs. A write fault to a text page looks like:

```
pte.valid == 1
pte.PTE_W == 0
pte.PTE_COW == 0   ← not COW!
```

The fault handler kills the process rather than attempting a copy. The `PTE_COW` check is what separates "deferred write" from "illegal write."

---

## Common Pitfalls

### 1. Forgetting to Update the Parent's PTE in uvmcopy()

```c
// Wrong: only set child's PTE as COW
mappages(child_pt, va, PGSIZE, pa, (flags | PTE_COW) & ~PTE_W);
// Parent's PTE still has PTE_W set!
```

```c
// Correct: update parent's PTE first
uint64 cow_flags = (flags | PTE_COW) & ~PTE_W;
*pte = PA2PTE(pa) | cow_flags;              // update parent PTE
mappages(child_pt, va, PGSIZE, pa, cow_flags);
```

Without this, the parent keeps writing to the shared page while the child reads stale data.

### 2. Decrementing ref count Before Copying

```c
// Wrong: page may be freed before the copy
kfree((void *)old_pa);
memmove(new_page, (void *)old_pa, PGSIZE);  // use-after-free!
```

```c
// Correct: copy first, then release
memmove(new_page, (void *)old_pa, PGSIZE);
kfree((void *)old_pa);
```

If another process holds the last reference and frees the page between your `kfree` and your `memmove`, you read garbage or panic.

### 3. Not Flushing TLB After Updating Parent's PTE

After modifying the parent's PTE in `uvmcopy()`, the TLB may still cache the old writable mapping. In xv6, `sfence.vma` handles this and is typically invoked during context switching — but a stale TLB entry can silently let writes bypass the write-protection, skipping the COW fault entirely.

### 4. Out-of-Memory During COW Fault

If `kalloc()` fails mid-fault, the process has nowhere to go. Return an error and kill it — don't silently return 0, which would let the process write through a mapping it doesn't own.

```c
void *new_page = kalloc();
if (new_page == 0) {
    p->killed = 1;
    return -1;
}
```

### 5. Race Condition on Reference Count

```
Process A reads ref count: 1
Process B reads ref count: 1       ← both think they're alone!
Process A skips copy, sets PTE_W
Process B skips copy, sets PTE_W   ← two processes now share a "writable" page!
```

Hold `pageref.lock` for the entire check-and-update sequence, not just the read.

### 6. Double-Freeing a Shared Page

```
Process A: count = 1 → 0, free page
Process B: count = 1 → 0, free page  ← double free!
```

The decrement and free decision must be atomic under the same lock.

### 7. uvmcopy() Partial Failure Cleanup

If `uvmcopy()` fails halfway (e.g., `mappages` runs out of memory), all pages already mapped to the child need to be released — including decrementing their ref counts. Passing `do_free = 1` to `uvmunmap` triggers `kfree` on each, which handles the ref count correctly.

```c
on_error:
    uvmunmap(child_pagetable, 0, mapped_pages, 1);  // calls kfree → decrements ref count
    return -1;
```
