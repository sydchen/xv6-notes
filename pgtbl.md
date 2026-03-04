# xv6 Page Tables

https://pdos.csail.mit.edu/6.1810/2025/labs/pgtbl.html

## Table of Contents
1. [Background: RISC-V Page Tables](#background-risc-v-page-tables)
2. [Lab Tasks Overview](#lab-tasks-overview)
3. [Task 1: Inspect a User-Process Page Table](#task-1-inspect-a-user-process-page-table)
4. [Task 2: Speed Up System Calls (USYSCALL)](#task-2-speed-up-system-calls-usyscall)
5. [Task 3: Print a Page Table (vmprint)](#task-3-print-a-page-table-vmprint)
6. [Task 4: Superpages](#task-4-superpages)
7. [Code Sketches](#code-sketches)
8. [Key Design Decisions](#key-design-decisions)
9. [Common Pitfalls](#common-pitfalls)

---

## Background: RISC-V Page Tables

### Three-Level Page Table Structure

RISC-V sv39 uses a three-level page table. Each level is a 4 KB page holding 512 page table entries (PTEs) of 8 bytes each:

```
Virtual Address (39 bits):
  [L2 index: 9 bits][L1 index: 9 bits][L0 index: 9 bits][page offset: 12 bits]

Translation walk:
  CR3 / satp → L2 page table
    [L2 index] → L1 page table
      [L1 index] → L0 page table
        [L0 index] → physical page
          + offset → final physical address
```

Each PTE holds a **physical page number (PPN)** and **flag bits**:

| Flag | Bit | Meaning |
|------|-----|---------|
| V | 0 | Valid — entry is present |
| R | 1 | Readable |
| W | 2 | Writable |
| X | 3 | Executable |
| U | 4 | User-accessible |
| G | 5 | Global mapping |
| A | 6 | Accessed |
| D | 7 | Dirty |

**Leaf vs. non-leaf:** If R, W, and X are all zero, the PTE points to the next level table. If any of R/W/X is set, it is a leaf — it directly maps to a physical page.

### Fixed-Address User Virtual Layout

xv6 maps several pages at the top of every user virtual address space (above the user program):

```
MAXVA
  TRAMPOLINE   (maps kernel's trampoline code, exec & ecall path)
  TRAPFRAME    (per-process, stores user registers on trap)
  USYSCALL     (new in this lab — shared read-only page)
  ...
  user stack / heap / text / data
0x0
```

### Superpages (2 MB Pages)

A normal leaf PTE sits at level 0, covering 4 KB. But a leaf PTE can also sit at **level 1**, covering 512 × 4 KB = **2 MB** — this is a superpage.

The hardware walks exactly the same way; the level-1 PTE simply has R/W/X set instead of pointing further down. No level-0 table is needed.

---

## Lab Tasks Overview

| Task | What you implement | Difficulty |
|------|--------------------|------------|
| Inspect page table | Analyze pgtbltest output, explain PTE meanings | Easy |
| USYSCALL | Shared read-only page for fast getpid() | Easy |
| vmprint() | Recursive page table printer | Easy |
| Superpages | 2 MB page allocation, copy, demotion | Moderate/Hard |

---

## Task 1: Inspect a User-Process Page Table

### Goal

Run the provided `pgtbltest` program and explain what each page table entry (PTE) represents — its virtual address, permission flags, and mapped physical page.

### What to Look For

A freshly started user process with `exec` has a predictable set of mappings:

```
page table 0x0000000087f6e000
 ..0: pte 0x21fda801 pa 0x87f6a000    ← L2 PTE pointing to L1 table
 .. ..0: pte 0x21fd9c01 pa 0x87f67000  ← L1 PTE pointing to L0 table
 .. .. ..0: pte 0x21fdac1f pa 0x87f6b000  ← text page (R, X, U, V)
 .. .. ..1: pte 0x21fda41f pa 0x87f69000  ← data page
 ...
 ..255: pte 0x...         pa 0x...     ← TRAMPOLINE / TRAPFRAME
```

A non-leaf PTE has R=W=X=0 (only V=1). A leaf PTE has at least one of R/W/X set. The `U` bit controls whether user mode can access the page.

### Key Observation

Virtual addresses are not necessarily contiguous in physical memory. The page table is the mapping layer that translates scattered physical frames into a contiguous virtual address space.

---

## Task 2: Speed Up System Calls (USYSCALL)

### The Problem

Every system call (even trivial ones like `getpid()`) requires a full round-trip: user → kernel → user. The cost is the trap overhead (saving/restoring registers, switching page tables, etc.).

### The Solution: A Shared Read-Only Page

Map a single page at a fixed virtual address (`USYSCALL`) that both the kernel can write to and user space can read from — **without a trap**:

```
Kernel side (proc.c):
  p->usyscall = kalloc();       // allocate physical page
  p->usyscall->pid = p->pid;   // fill in the PID
  mappages(pagetable, USYSCALL, PGSIZE,
           (uint64)p->usyscall,
           PTE_R | PTE_U);      // user: read only, kernel: r/w

User side (ulib.c):
  int getpid(void) {
      struct usyscall *u = (struct usyscall *)USYSCALL;
      return u->pid;  // direct memory read, no trap!
  }
```

No `ecall` is needed on the read path. The kernel keeps `p->usyscall->pid` updated when process metadata changes.

### Lifecycle

| Event | Action |
|-------|--------|
| `allocproc()` | `kalloc()` the page, set `pid` |
| `proc_pagetable()` | `mappages(USYSCALL, PTE_R|PTE_U)` |
| `proc_freepagetable()` | `uvmunmap(USYSCALL, ...)` |
| `freeproc()` | `kfree(p->usyscall)` |

The USYSCALL mapping must be removed before its physical page is freed; otherwise the VA may point to recycled memory.

### Which Other Syscalls Could Benefit?

This pattern fits syscalls that expose **read-only, low-churn kernel state** (`getppid()`, `uptime()`, CPU count, etc.), where kernel can proactively publish values.

---

## Task 3: Print a Page Table (vmprint)

### Goal

Implement `vmprint(pagetable_t pagetable)` that recursively prints every valid PTE in a three-level page table, with indentation showing the level:

```
page table 0x0000000087f6e000
 ..0: pte 0x... pa 0x...        ← level 2 (one leading "..")
 .. ..0: pte 0x... pa 0x...     ← level 1 (two leading "..")
 .. .. ..0: pte 0x... pa 0x...  ← level 0 / leaf
```

### Algorithm

```c
void vmprintwalk(pagetable_t pt, int level) {
    for (int i = 0; i < 512; i++) {
        pte_t pte = pt[i];
        if (!(pte & PTE_V))
            continue;

        // Print indentation: (level+1) pairs of " .."
        for (int j = 0; j <= level; j++)
            printf(" ..");
        uint64 pa = PTE2PA(pte);
        printf("%p: pte %lx pa %lx\n", &pt[i], pte, pa);

        // Non-leaf: R=W=X=0, recurse into child page table
        if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)
            vmprintwalk((pagetable_t)pa, level + 1);
    }
}

void vmprint(pagetable_t pagetable) {
    printf("page table %p\n", pagetable);
    vmprintwalk(pagetable, 0);
}
```

### When is it Called?

`vmprint` is typically called on the first `exec` for inspection, and is also useful during panic-time debugging.

---

## Task 4: Superpages

### Motivation

Mapping 2 MB with normal pages needs one L0 table plus 512 L0 PTEs. A superpage replaces that with one L1 leaf PTE, reducing table overhead and TLB pressure.

### The Plan

1. **Reserve** a pool of 2 MB-aligned physical chunks at boot (`kinit`).
2. **Allocate** from this pool (`superalloc`) when `uvmalloc` gets a 2 MB-aligned address that needs ≥ 2 MB more.
3. **Map** as a level-1 leaf PTE (`set_superpage`).
4. **Copy** an entire superpage in `uvmcopy` (for `fork`).
5. **Free** entire superpages in `uvmunmap`.
6. **Demote** a superpage into 512 normal pages if only part of it is freed.

### Superpage Memory Pool

At boot, before adding memory to the normal free list, the kernel carves out a fixed number of 2 MB-aligned regions:

```c
void kinit() {
    // find first 2MB-aligned address after kernel 'end'
    uint64 aligned_start = SUPERPGROUNDUP((uint64)end);

    // reserve up to 16 superpages (32 MB total)
    for (each 2MB region starting at aligned_start) {
        add to super_kmem.superlist;
    }

    // give remaining normal pages to kmem.freelist
    freerange(end, aligned_start);
    freerange(aligned_start + reserved_size, PHYSTOP);
}
```

### Setting a Superpage Mapping

A superpage leaf PTE lives at level 1, not level 0. To install one:

```c
int set_superpage(pagetable_t pagetable, uint64 va, uint64 pa, int perm) {
    // va and pa must both be 2MB-aligned
    pte_t *pte2 = &pagetable[PX(2, va)];       // L2 entry

    // allocate L1 table if missing
    if (!(*pte2 & PTE_V)) {
        pagetable_t level1 = kalloc();
        *pte2 = PA2PTE(level1) | PTE_V;
    }

    pagetable_t level1 = PTE2PA(*pte2);
    pte_t *pte1 = &level1[PX(1, va)];           // L1 entry

    // set as leaf (R/W/X bits present → hardware treats as leaf)
    *pte1 = PA2PTE(pa) | perm | PTE_V;

    // No L0 table allocated — that's the point!
    return 0;
}
```

### Detecting a Superpage

To tell if a virtual address is backed by a superpage:

```c
int is_superpage_pte(pagetable_t pagetable, uint64 va) {
    pte_t *pte2 = &pagetable[PX(2, va)];
    if (!(*pte2 & PTE_V) || PTE_LEAF(*pte2))
        return 0;   // L2 is missing or itself a leaf

    pagetable_t level1 = PTE2PA(*pte2);
    pte_t *pte1 = &level1[PX(1, va)];

    // A valid leaf at L1 == superpage
    return (*pte1 & PTE_V) && PTE_LEAF(*pte1);
}
```

### Demotion: Splitting a Superpage

When only part of a superpage needs to be freed, you can't just remove the L1 leaf PTE — the rest of the data would disappear too. Instead, **demote** it:

```c
int demote_superpage(pagetable_t pagetable, uint64 va, pte_t *superpte) {
    uint64 superpa = PTE2PA(*superpte);
    uint flags = PTE_FLAGS(*superpte);

    // allocate a new L0 page table
    pagetable_t level0 = kalloc();

    // copy each 4KB chunk of the superpage into its own page
    for (int i = 0; i < 512; i++) {
        char *mem = kalloc();
        memmove(mem, (char*)(superpa + i * PGSIZE), PGSIZE);
        level0[i] = PA2PTE(mem) | flags;
    }

    // replace the L1 leaf PTE with a pointer to the new L0 table
    *superpte = PA2PTE(level0) | PTE_V;   // PTE_V only → non-leaf

    // free the original 2MB physical chunk
    superfree((void*)superpa);
    return 0;
}
```

After demotion, the address range looks exactly like normal 4 KB mappings.

---

## Code Sketches

### 1. usyscall struct and mapping

```c
// kernel/memlayout.h
struct usyscall {
    int pid;   // process ID, readable from user space
};

// kernel/proc.c — allocproc()
p->usyscall = (struct usyscall *)kalloc();
p->usyscall->pid = p->pid;

// kernel/proc.c — proc_pagetable()
mappages(pagetable, USYSCALL, PGSIZE,
         (uint64)p->usyscall,
         PTE_R | PTE_U);

// kernel/proc.c — proc_freepagetable()
uvmunmap(pagetable, USYSCALL, 1, 0);   // unmap but don't free

// kernel/proc.c — freeproc()
kfree((void*)p->usyscall);
p->usyscall = 0;
```

### 2. vmprint() — hierarchical page table dump

```c
void vmprintwalk(pagetable_t pt, int level) {
    for (int i = 0; i < 512; i++) {
        pte_t pte = pt[i];
        if (!(pte & PTE_V)) continue;

        for (int j = 0; j <= level; j++) printf(" ..");
        uint64 pa = PTE2PA(pte);
        printf("%p: pte %lx pa %lx\n", &pt[i], pte, pa);

        if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)
            vmprintwalk((pagetable_t)pa, level + 1);
    }
}

void vmprint(pagetable_t pagetable) {
    printf("page table %p\n", pagetable);
    vmprintwalk(pagetable, 0);
}
```

### 3. superalloc / superfree

```c
struct { struct spinlock lock; struct run *superlist; } super_kmem;

void *superalloc(void) {
    acquire(&super_kmem.lock);
    struct run *r = super_kmem.superlist;
    if (r) super_kmem.superlist = r->next;
    release(&super_kmem.lock);
    if (r) memset(r, 5, SUPERPGSIZE);   // fill with junk
    return (void *)r;
}

void superfree(void *pa) {
    // validate alignment and range
    memset(pa, 1, SUPERPGSIZE);
    struct run *r = (struct run *)pa;
    acquire(&super_kmem.lock);
    r->next = super_kmem.superlist;
    super_kmem.superlist = r;
    release(&super_kmem.lock);
}
```

### 4. uvmalloc — try superpage first

```c
for (a = oldsz; a < newsz; a += sz) {
    // Attempt superpage if 2MB-aligned and ≥2MB remaining
    if ((a % SUPERPGSIZE) == 0 && (newsz - a) >= SUPERPGSIZE) {
        sz = SUPERPGSIZE;
        mem = superalloc();
        if (mem != 0) {
            memset(mem, 0, sz);
            if (set_superpage(pagetable, a, (uint64)mem,
                              PTE_V|PTE_R|PTE_W|PTE_U|xperm) != 0) {
                superfree(mem);
                return 0;
            }
            continue;
        }
        // superalloc failed → fall through to normal 4KB allocation
    }
    sz = PGSIZE;
    // ... normal kalloc + mappages ...
}
```

### 5. uvmunmap — handle superpages

```c
for (a = va; a < va + npages * PGSIZE; a += sz) {
    sz = PGSIZE;
    uint64 base_va = SUPERPGROUNDDOWN(a);

    if (is_superpage_pte(pagetable, base_va)) {
        pte_t *pte1 = /* get L1 PTE for base_va */;
        uint64 end_va = va + npages * PGSIZE;

        if (a == base_va && end_va >= base_va + SUPERPGSIZE) {
            // unmapping entire superpage
            sz = SUPERPGSIZE;
            if (do_free) superfree((void *)PTE2PA(*pte1));
            *pte1 = 0;
            continue;
        } else {
            // partial unmap: demote first, then fall through
            if (do_free)
                demote_superpage(pagetable, base_va, pte1);
            else { *pte1 = 0; sz = SUPERPGSIZE; continue; }
        }
    }

    // handle as normal 4KB page...
}
```

---

## Key Design Decisions

### 1. USYSCALL: Read-Only from User, Read-Write from Kernel

The `PTE_U` flag without `PTE_W` ensures user code can read the shared page but cannot write it. Only the kernel (which bypasses `PTE_U` checks in supervisor mode) can update the `pid` field.

This is a deliberate security boundary: the fast path is fast precisely because the kernel controls what's in the page.

### 2. vmprint: Distinguish Leaf vs. Non-Leaf by R/W/X Bits

In RISC-V, the only way to know if a PTE is a leaf is to check whether any of R, W, or X is set. A PTE with `V=1` but `R=W=X=0` points to the next-level table — never stop recursing based on level alone (a superpage leaf lives at level 1, not level 0).

### 3. Superpage Pool: Pre-Allocated at Boot, Not On-Demand

Dynamic 2 MB-aligned allocation from the normal free list is awkward (fragmentation + alignment). Instead, the kernel **reserves** fixed 2 MB chunks at boot and keeps them in `superlist`.

Trade-off: pool size is fixed (16 superpages = 32 MB). Once exhausted, `uvmalloc` falls back to normal 4 KB mappings.

### 4. Demotion: Copy Data Before Freeing the Superpage

When only part of a 2 MB region is unmapped, the remaining data must survive. Demotion does:
1. Allocates 512 separate 4 KB pages
2. Copies each 4 KB chunk from the superpage
3. Wires up a new L0 page table
4. **Then** frees the original 2 MB chunk

Order matters: freeing first would make the copy read freed memory.

### 5. Superpage Alignment: Both VA and PA Must Be 2 MB Aligned

For a level-1 leaf mapping, both VA and PA must be 2 MB-aligned. Otherwise offset interpretation inside the superpage is wrong.

`SUPERPGROUNDUP` / `SUPERPGROUNDDOWN` enforce this before superpage operations.

---

## Common Pitfalls

### 1. Forgetting to Unmap USYSCALL in proc_freepagetable

If the physical page is freed first, later page-table walking can dereference stale mappings.

**Correct order:**
```c
proc_freepagetable(p->pagetable, p->sz);  // unmap USYSCALL here...
p->pagetable = 0;
// ...
kfree(p->usyscall);                        // ...then free the physical page
```

Call `uvmunmap(pagetable, USYSCALL, 1, 0)` inside `proc_freepagetable`, not inside `freeproc`.

### 2. vmprint: Recursing into Leaf PTEs

A level-1 superpage entry is still a leaf (`R/W/X != 0`). Recursing by level number alone can treat data pages as page tables.

**Wrong:**
```c
if (level < 2)   // always recurse at levels 0 and 1
    vmprintwalk((pagetable_t)pa, level + 1);
```

**Correct:**
```c
if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)   // non-leaf → recurse
    vmprintwalk((pagetable_t)pa, level + 1);
```

### 3. Superpage Demotion: Do Not Free the Superpage Before Copying

The superpage is the **source** of the copy. Freeing it before all 512 chunks are copied produces data corruption.

### 4. Partial Superpage Unmap When do_free=0

When `uvmunmap` is called with `do_free=0` (e.g., during `exec`'s cleanup of the old address space), you should not demote the superpage — just clear the PTE. Demoting without freeing would allocate 512 new 4 KB pages and the superpage physical memory would leak.

### 5. is_superpage_pte: Check the Base Address, Not the Current Address

When iterating through a range, addresses inside one 2 MB window belong to the same superpage. Round down before `is_superpage_pte`:

```c
uint64 base_va = SUPERPGROUNDDOWN(a);
if (is_superpage_pte(pagetable, base_va)) { ... }
```

Using non-aligned `a` directly may check wrong L2/L1 indices.

### 6. USYSCALL Permission Bits: PTE_R | PTE_U, Not PTE_W

The user must be able to **read** the page (to get the PID) but must **not write** it. Adding `PTE_W` would let user code corrupt kernel-managed data.

```c
// Correct:
mappages(pagetable, USYSCALL, PGSIZE, (uint64)p->usyscall, PTE_R | PTE_U);

// Wrong (writable from user):
mappages(pagetable, USYSCALL, PGSIZE, (uint64)p->usyscall, PTE_R | PTE_W | PTE_U);
```

---

## How the Pieces Fit Together

```
sbrk(2MB)
  → sys_sbrk → growproc(2MB)
      → uvmalloc(pagetable, oldsz, newsz, xperm)
          if (2MB-aligned && 2MB remaining):
              mem = superalloc()   ← from boot-reserved pool
              set_superpage(pagetable, va, pa, flags)
                  ← L2 non-leaf → L1 leaf PTE (no L0!)
          else:
              kalloc() + mappages() as usual

fork()
  → uvmcopy(old, new, sz)
      if is_superpage_pte(old, i):
          superalloc() + memmove(2MB) + set_superpage(new, ...)
      else:
          kalloc() + memmove(4KB) + mappages()

sbrk(-PGSIZE) on a superpage boundary
  → uvmunmap(pagetable, va, 1, do_free=1)
      is_superpage_pte? partial unmap detected
      → demote_superpage():
           allocate L0 table + 512×kalloc
           copy each 4KB from superpage
           replace L1 leaf PTE with L0 table pointer
           superfree(original 2MB page)
      → normal 4KB unmap logic handles the rest
```
