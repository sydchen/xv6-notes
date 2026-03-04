# xv6 Memory-Mapped Files

https://pdos.csail.mit.edu/6.1810/2025/labs/mmap.html

## Table of Contents
1. [What Are Memory-Mapped Files?](#what-are-memory-mapped-files)
2. [Core Concepts](#core-concepts)
3. [Implementation Architecture](#implementation-architecture)
4. [Pseudo Code](#pseudo-code)
5. [Key Design Decisions](#key-design-decisions)
6. [Common Pitfalls and Fixes](#common-pitfalls-and-fixes)
7. [Performance Optimization Ideas](#performance-optimization-ideas)

---

## What Are Memory-Mapped Files?

The idea of memory-mapped files is simple: map file contents into a process's virtual address space, then read/write the file as if you were accessing memory directly.

### Core Benefits

1. **Simpler I/O**: In many cases, load/store can replace `read/write`
2. **Shared memory**: Multiple processes can map the same file and share data
3. **On-demand loading**: Pages are loaded only when touched (lazy loading)
4. **Better performance**: Fewer user/kernel mode switches

### Basic Usage Example

```c
// Open file
int fd = open("data.txt", O_RDWR);

// Map file into memory
void *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, 0);

// Access file like memory
char *data = (char *)addr;
data[0] = 'A';  // Written back to file if MAP_SHARED

// Unmap
munmap(addr, 4096);
```

---

## Core Concepts

### 1. VMA (Virtual Memory Area)

A VMA describes one continuous virtual memory region:

```c
struct vma {
  int used;              // Whether this slot is in use
  uint64 addr;           // Start virtual address
  uint64 length;         // Length (bytes)
  int prot;              // Permissions (PROT_READ | PROT_WRITE)
  int flags;             // MAP_SHARED or MAP_PRIVATE
  struct file *file;     // Backing file
  uint64 offset;         // File offset
};
```

### 2. Lazy Allocation

When `mmap` is called, xv6 **does not allocate physical memory immediately**. Instead it:
- Creates a VMA record only
- Triggers a page fault on first access
- Lets the page-fault handler allocate a page and load file content

**Why lazy allocation?**
- `mmap` can return in near O(1)
- Physical memory is used only for touched pages
- Works better for large file mappings

### 3. MAP_SHARED vs MAP_PRIVATE

| Property | MAP_SHARED | MAP_PRIVATE |
|----------|------------|-------------|
| Write visibility | Written back to file | Not written back |
| Use case | Inter-process sharing, persistence | Private copy, temporary edits |
| Implementation | Write back on munmap/exit | Ignore write-back entirely |

---

## Implementation Architecture

### System Components

```
User Space
  ↓ mmap() / munmap()
Kernel System Calls (sysfile.c)
  ↓ VMA management
Process Structure (proc.h)
  ↓ page fault
Page Fault Handler (vm.c)
  ↓ file I/O
File System (fs.c)
```

### Main Flows

**mmap flow:**
```
mmap() → create VMA → return virtual address
       → increase file reference count
       → do not allocate physical pages yet
```

**First-access flow:**
```
access address → page fault → check VMA
               → allocate physical page
               → read file content (readi)
               → create page-table mapping
               → return to user space
```

**munmap flow:**
```
munmap() → find target VMA
         → write back changes (if MAP_SHARED)
         → unmap page table entries (uvmunmap)
         → update/remove VMA
         → decrease file reference count
```

---

## Pseudo Code

### 1. mmap System Call

```c
uint64
sys_mmap(uint64 addr, uint64 length, int prot, int flags, int fd, uint64 offset)
{
    // Validate arguments and file permissions
    validate_params(length, offset, fd, prot, flags);

    struct proc *p = myproc();

    // Find free VMA slot
    struct vma *v = find_free_vma(p);

    // Allocate virtual address (grow upward from process.sz)
    length = PGROUNDUP(length);
    addr = p->sz;
    p->sz += length;

    // Initialize VMA (no physical memory allocation here)
    v->addr = addr;
    v->length = length;
    v->prot = prot;
    v->flags = flags;
    v->file = filedup(f);  // Important: increase file ref count

    return addr;
}
```

### 2. Page Fault Handler (vmfault)

```c
void *
vmfault(pagetable_t pagetable, uint64 va, int is_write)
{
    va = PGROUNDDOWN(va);

    // Check whether va belongs to a VMA
    struct vma *v = find_vma_for_address(myproc(), va);
    if (v == 0)
        return (void *)-1;  // FAILURE

    // Permission check (writing to read-only mapping should fail)
    if (is_write && !(v->prot & PROT_WRITE))
        return (void *)-1;

    // Allocate and zero-fill a physical page
    void *page = kalloc();
    memset(page, 0, PGSIZE);

    // Read content from backing file
    uint64 offset = v->offset + (va - v->addr);
    struct inode *ip = v->file->ip;
    ilock(ip);
    readi(ip, 0, (uint64)page, offset, PGSIZE);
    iunlock(ip);

    // Install page-table mapping
    int perm = PTE_U | prot_to_pte(v->prot);
    mappages(pagetable, va, PGSIZE, (uint64)page, perm);

    return page;
}
```

### 3. munmap System Call

```c
int
sys_munmap(uint64 addr, uint64 length)
{
    struct proc *p = myproc();

    // Find target VMA
    struct vma *v = find_vma_for_address(p, addr);

    // MAP_SHARED: write back mapped pages
    if (v->flags & MAP_SHARED) {
        for (uint64 pg = addr; pg < addr + length; pg += PGSIZE) {
            if (page_is_mapped(p->pagetable, pg)) {
                uint64 offset = v->offset + (pg - v->addr);
                begin_op();
                ilock(v->file->ip);
                writei(v->file->ip, 1, pg, offset, PGSIZE);
                iunlock(v->file->ip);
                end_op();
            }
        }
    }

    // Unmap and free physical pages
    uvmunmap(p->pagetable, addr, length / PGSIZE, 1);

    // Update or remove VMA
    if (addr == v->addr && length == v->length) {
        // unmapping entire region
        v->used = 0;
        fileclose(v->file);
    } else if (addr == v->addr) {
        // unmapping from start
        v->addr += length;
        v->length -= length;
    } else {
        // unmapping from end
        v->length -= length;
    }

    return 0;  // SUCCESS
}
```

### 4. VMA Handling in Fork

```c
int
fork(void)
{
    // ... create child process, copy page table, etc. ...

    struct proc *p = myproc();
    struct proc *np = /* newly allocated child proc */;

    // Copy all VMAs (without copying physical pages)
    for (int i = 0; i < NVMA; i++) {
        struct vma *v = &p->vmas[i];
        if (v->used) {
            np->vmas[i] = *v;                          // copy metadata
            np->vmas[i].file = filedup(v->file);       // increase file ref count
        }
    }

    // Child loads its own pages through page faults
}
```

**Note**: Parent and child do not share physical pages. This keeps the implementation simple and avoids COW complexity.

### 5. VMA Cleanup in Exit

```c
void
exit(int status)
{
    // ... close file descriptors, cwd, etc. ...

    struct proc *p = myproc();

    // Clean up all VMAs
    for (int i = 0; i < NVMA; i++) {
        struct vma *v = &p->vmas[i];
        if (v->used) {
            // MAP_SHARED: write back all mapped pages
            if (v->flags & MAP_SHARED)
                writeback_all_pages(v);

            // Unmap and free physical pages
            uvmunmap(p->pagetable, v->addr, v->length / PGSIZE, 1);

            // Close file (decrease reference count)
            fileclose(v->file);
            v->used = 0;
        }
    }
}
```

---

## Key Design Decisions

### 1. Virtual Address Allocation Strategy

**Choice: grow upward from `process.sz`**

```
+-------------------+  ← MAXVA
| TRAMPOLINE        |
+-------------------+  ← MAXVA - PGSIZE
| TRAPFRAME         |
+-------------------+  ← MAXVA - 2*PGSIZE
|                   |
|   (unused area)   |
|                   |
+-------------------+  ← process.sz (grows dynamically)
| mmap region 2     |  ← second mmap
+-------------------+
| mmap region 1     |  ← first mmap
+-------------------+
| HEAP              |  ← sbrk() allocation
+-------------------+
| DATA              |
+-------------------+
| TEXT              |
+-------------------+  ← 0x0
```

**Pros:**
- Easy to implement; consistent with heap growth direction
- No separate virtual-address allocator needed
- Simpler conflict handling

**Cons:**
- Reduces available heap growth space
- Calling `sbrk` after `mmap` can easily conflict

**Alternative:**
- Grow downward from TRAPFRAME (more complex, but avoids heap interference)

### 2. VMA Data Structure

**Choice: fixed-size array (16 slots)**

```c
#define NVMA 16
struct proc {
    // ...
    struct vma vmas[NVMA];
};
```

**Pros:**
- Simple; no dynamic allocation required
- Bounded and predictable traversal cost
- Easier to debug

**Cons:**
- Caps number of concurrent mappings
- Wastes some memory on unused slots

**Alternatives:**
- Linked list: dynamic, but needs memory allocation
- Red-black tree: faster lookup, but overkill for this lab

### 3. Where to Do Lazy Allocation

**Why load on page fault instead of at `mmap` time?**

| Approach | mmap latency | Memory usage | Supports large files |
|----------|--------------|--------------|----------------------|
| Eager (load at mmap) | O(n) | High | ✗ |
| Lazy (load on fault) | O(1) | Low | ✓ |

**Practical difference:**
```c
// Map a 1GB file
void *addr = mmap(NULL, 1GB, ...);
// Eager: allocate 256K pages (can take seconds)
// Lazy: return immediately (microseconds)

// Access only the first page
char c = ((char*)addr)[0];
// Eager: already loaded
// Lazy: one page fault, load one page
```

### 4. MAP_SHARED Write-Back Timing

**Choice: write back on `munmap` and `exit`**

**Why not write back immediately after each page fault/write?**
1. Performance: writing every change is too expensive
2. Consistency: the same page may be modified many times
3. Simplicity: no need to track dirty pages in a fine-grained way

**Dirty-bit optimization (not implemented):**
```c
if pte & PTE_DIRTY:
    write_back_to_file()
else:
    skip  // unchanged page
```

**Current implementation:**
- Write back **all** mapped pages (whether modified or not)
- Trade performance for simplicity and correctness

### 5. Page Sharing Strategy in Fork

**Choice: do not share physical pages**

**Two approaches:**

| Approach | Complexity | Memory usage | Consistency |
|----------|------------|--------------|-------------|
| Shared pages (COW) | High | Low | Complex |
| No sharing | Low | High | Simple |

**Implementation detail:**
```c
// Approach 1: shared pages (not used)
fork() {
    // Mark parent pages read-only
    // Child shares same physical pages
    // On write, trigger COW
}

// Approach 2: no sharing (used)
fork() {
    // Copy only VMA metadata
    // Child loads its own pages via page fault
}
```

---

## Common Pitfalls and Fixes

### 1. Deadlock from Wrong Lock Ordering

**Incorrect example:**
```c
// Incorrect: may deadlock
ilock(ip);
begin_op();
writei(...);
end_op();
iunlock(ip);
```

**Correct order:**
```c
// Correct: begin_op must be outside
begin_op();
ilock(ip);
writei(...);
iunlock(ip);
end_op();
```

**Reason:** `begin_op` may block waiting for journal space. Holding an inode lock while waiting can deadlock.

### 2. File Reference Count Bug

**Incorrect example:**
```c
// Incorrect: no ref count increment
v->file = f;  // direct assignment
```

**Correct way:**
```c
// Correct: use filedup
v->file = filedup(f);  // increment ref count
```

**Consequence:** Without incrementing reference count, `close(fd)` may free the file while VMA still points to it (dangling pointer).

**Test case:**
```c
int fd = open("file.txt", O_RDWR);
void *addr = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);
close(fd);  // without filedup, file may be freed here

// Accessing addr may crash (dangling pointer)
char c = ((char*)addr)[0];  // crash or garbage
```

### 3. Permission Check at the Wrong Time

**Incorrect example:**
```c
// Incorrect: check after allocation
mem = kalloc();
if (!(vma->prot & PROT_WRITE))
    return 0;  // mem leaked!
```

**Correct way:**
```c
// Correct: check before allocation
if (!write && !(vma->prot & PROT_WRITE))
    return 0;
mem = kalloc();
```

### 4. Ignoring Partial Last Page of a File

**Problem:** File size may not be a multiple of page size.

**Incorrect example:**
```c
// Incorrect: may write past file end
writei(ip, 0, pa, file_offset, PGSIZE);
```

**Correct way:**
```c
// Correct: cap write size
max_write = PGSIZE;
if (file_offset + max_write > ip->size)
    max_write = ip->size - file_offset;

if (max_write > 0)
    writei(ip, 0, pa, file_offset, max_write);
```

**Example:**
```
File size: 4100 bytes
Mapping size: 8192 bytes (2 pages)

Page 0: [0, 4096)     → full page, write back 4096 bytes ✓
Page 1: [4096, 8192)  → partial page, write back only 4 bytes ✓
```

### 5. Forgetting to Update `process.sz`

**Incorrect example:**
```c
// Incorrect: address space not extended
addr = p->sz;
v->addr = addr;
v->length = length;
// missing p->sz update
```

**Correct way:**
```c
// Correct: extend address space
addr = p->sz;
p->sz += length;  // required
v->addr = addr;
v->length = length;
```

**Consequence:** Later `sbrk` or `mmap` may overwrite this mapped region.

### 6. Forgetting to Zero-Fill New Pages

**Incorrect example:**
```c
// Incorrect: no zero-fill
mem = kalloc();
readi(ip, 0, mem, offset, PGSIZE);
```

**Correct way:**
```c
// Correct: zero-fill first
mem = kalloc();
memset(mem, 0, PGSIZE);  // required
readi(ip, 0, mem, offset, PGSIZE);
```

**Reason:**
- `kalloc` may return pages containing old data
- If file size is smaller than one page, `readi` fills only part of the page
- Unread bytes should be zero (Unix expected behavior)

---

## Performance Optimization Ideas

xv6 prioritizes simplicity over performance, but these optimizations are possible:

### 1. Dirty-Bit Tracking

**Current:** Write back all mapped pages  
**Optimized:** Write back only dirty pages

```c
// Check PTE dirty bit
if (pte & PTE_D) {  // PTE_D = dirty bit
    writei(...);
    *pte &= ~PTE_D;  // clear dirty bit
}
```

**Benefit:** Less unnecessary disk I/O

### 2. Batched Write-Back

**Current:** One transaction per page (`begin_op/end_op`)  
**Optimized:** Batch multiple pages in one transaction

```c
begin_op();
ilock(ip);
for each dirty page:
    writei(...);
iunlock(ip);
end_op();
```

**Benefit:** Lower journaling overhead

### 3. VMA Merging

**Current:** Adjacent mappings consume multiple VMAs  
**Optimized:** Merge adjacent compatible mappings automatically

```c
// Check whether new mapping can merge with an existing VMA
for each vma:
    if vma.addr + vma.length == new_addr and
       vma.file == new_file and
       vma.prot == new_prot and
       vma.flags == new_flags:
        // merge instead of creating a new VMA
        vma.length += new_length
        return vma.addr
```

**Benefit:** Use fewer VMA slots, support more mappings

### 4. Readahead

**Current:** Load only the faulting page  
**Optimized:** Predictively load neighboring pages

```c
vmfault(va) {
    // Load current page
    load_page(va);

    // Readahead next pages
    if (sequential_access_pattern) {
        for i = 1 to READAHEAD_SIZE:
            load_page(va + i * PGSIZE);
    }
}
```

**Benefit:** Fewer page faults

### Manual Test Cases

```c
// Test 1: Basic functionality
int fd = open("README", O_RDONLY);
char *p = mmap(0, 100, PROT_READ, MAP_PRIVATE, fd, 0);
printf("%c", p[0]);  // should succeed
munmap(p, 100);

// Test 2: Write to read-only mapping (should fail)
p = mmap(0, 100, PROT_READ, MAP_PRIVATE, fd, 0);
p[0] = 'X';  // should trigger page fault -> process killed

// Test 3: Accessible after closing fd
int fd = open("file.txt", O_RDWR);
char *p = mmap(0, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);  // close fd
p[0] = 'A';  // still accessible (because of filedup)
munmap(p, 4096);

// Test 4: MAP_SHARED write-back
fd = open("output.txt", O_RDWR);
p = mmap(0, 100, PROT_WRITE, MAP_SHARED, fd, 0);
p[0] = 'H';
p[1] = 'i';
munmap(p, 100);  // write back to file
// Re-open file, should see "Hi"
```
