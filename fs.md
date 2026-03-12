# xv6 File System

https://pdos.csail.mit.edu/6.1810/2025/labs/fs.html

## Table of Contents
1. [xv6 File System Overview](#xv6-file-system-overview)
2. [Lab Tasks Overview](#lab-tasks-overview)
3. [Task 1: Large Files (Doubly-Indirect Blocks)](#task-1-large-files-doubly-indirect-blocks)
4. [Task 2: Symbolic Links](#task-2-symbolic-links)
5. [Code Sketches](#code-sketches)
6. [Design Decisions](#design-decisions)
7. [Common Pitfalls](#common-pitfalls)

---

## xv6 File System Overview

### On-Disk Layout

xv6 organizes a disk as a sequence of fixed-size blocks (1024 bytes each):

```
| boot | super | log | inodes | bitmap | data blocks ... |
```

- **Superblock**: file system metadata (size, number of inodes, etc.)
- **Log**: write-ahead log for crash recovery
- **Inode blocks**: array of on-disk inodes (`struct dinode`)
- **Bitmap**: one bit per data block, tracks free/used
- **Data blocks**: actual file and directory content

### Inode and Block Mapping

Every file is represented by an **inode**, which stores metadata and a list of data block addresses. The challenge: inodes have a fixed size on disk, so they can't store an unbounded list of addresses. xv6 solves this with **indirect blocks**:

```
Before this lab (original):
  addrs[0..10]   → 11 direct data blocks
  addrs[11]      → singly-indirect block
                    └─ 256 block addresses → 256 more data blocks
  Max: 11 + 256 = 267 blocks ≈ 267 KB

After this lab:
  addrs[0..10]   → 11 direct data blocks
  addrs[11]      → singly-indirect block → 256 data blocks
  addrs[12]      → doubly-indirect block
                    └─ 256 first-level pointers
                        └─ each → 256 data blocks
  Max: 11 + 256 + 256×256 = 65,803 blocks ≈ 64 MB
```

### The `bmap()` Function

`bmap(ip, bn)` translates a **logical block number** (0, 1, 2, …) within a file into a **disk block number**. It allocates blocks on demand if they don't exist yet.

`bmap()` is the only place that really needs to understand the indirect layout. Most read/write paths just rely on it.

---

## Lab Tasks Overview

| Task | What changes | Key concept |
|------|-------------|-------------|
| Large files | Add doubly-indirect block to `bmap()` and `itrunc()` | Two-level index arithmetic |
| Symbolic links | New `symlink()` syscall + modify `sys_open()` | Inode lock protocol during traversal |

These two tasks are mostly independent.

---

## Task 1: Large Files (Doubly-Indirect Blocks)

### The Problem

Original `addrs` array: `addrs[NDIRECT+1]` where `NDIRECT=12`.
- `addrs[0..11]`: 12 direct blocks
- `addrs[12]`: singly-indirect block

To add doubly-indirect support **without changing the on-disk inode size**, we trade one direct block for the new level:
- Reduce `NDIRECT` from 12 → 11
- `addrs[11]`: singly-indirect (unchanged role, just shifted)
- `addrs[12]`: doubly-indirect (new)
- `addrs` array size: `NDIRECT+2` = 13 entries (same total bytes as before since we decreased NDIRECT by 1)

### Two-Level Index Arithmetic

For a logical block number `bn` that falls in the doubly-indirect range:

```
bn -= NINDIRECT          // subtract singly-indirect range first

idx1 = bn / NINDIRECT    // which first-level slot (0..255)
idx2 = bn % NINDIRECT    // which second-level slot within that (0..255)

disk layout:
  addrs[NDIRECT+1]       → doubly-indirect root block
    [idx1]               → first-level indirect block
      [idx2]             → actual data block
```

`NINDIRECT = BSIZE / sizeof(uint) = 1024 / 4 = 256`

### itrunc() Teardown Order

When truncating a file, blocks must be freed **inside-out** — data blocks first, then the blocks that pointed to them:

```
For the doubly-indirect block:
  1. For each first-level pointer [j]:
       a. Read the first-level block
       b. For each second-level pointer [k]: free data block
       c. Free the first-level block itself
  2. Free the doubly-indirect root block
  3. Set addrs[NDIRECT+1] = 0
```

Freeing the root before its children would lose the pointers to the children — a leak.

---

## Task 2: Symbolic Links

### What a Symlink Is

A symbolic link is a file whose content is a **path string** pointing to another file. Unlike hard links (which directly share an inode), symlinks are independent files with their own inode of type `T_SYMLINK`.

```
hard link:    direntry "a" ──► inode 17 (data)
              direntry "b" ──► inode 17 (same inode)

symlink:      direntry "link" ──► inode 42 (T_SYMLINK, content="/path/to/target")
              direntry "target" ──► inode 17 (data)
```

### New Additions

```c
// stat.h
#define T_SYMLINK 4

// fcntl.h
#define O_NOFOLLOW 0x800   // open the link itself, don't follow
```

### sys_symlink()

Creating a symlink:
1. Create a new inode of type `T_SYMLINK` at `path`
2. Write the `target` string into its first data block
3. That's it — no special format, just a raw path string

### Following Symlinks in sys_open()

When `open()` encounters a `T_SYMLINK` inode:
1. Read the target path from the inode's data
2. Release the current inode (`iunlockput`)
3. Resolve the new path with `namei()`
4. Lock the new inode
5. Repeat if the new inode is also a symlink (recursive resolution)
6. Stop if depth ≥ 10 (cycle detection)

Hold the inode lock when reading `ip->type`, and release it before calling `namei()`.

---

## Code Sketches

### 1. Modified inode structure

```c
// Before
#define NDIRECT 12
uint addrs[NDIRECT+1];   // 13 entries: 12 direct + 1 singly-indirect

// After
#define NDIRECT 11
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT*NINDIRECT)
uint addrs[NDIRECT+2];   // 13 entries: 11 direct + 1 singly + 1 doubly
//          ↑ still 13 entries total — on-disk inode size unchanged
```

### 2. bmap() — doubly-indirect case

```c
uint bmap(struct inode *ip, uint bn) {
    uint addr;
    struct buf *bp;
    uint *a;

    // Direct blocks: addrs[0..NDIRECT-1]
    if (bn < NDIRECT) { ... }
    bn -= NDIRECT;

    // Singly-indirect: addrs[NDIRECT]
    if (bn < NINDIRECT) { ... }
    bn -= NINDIRECT;

    // Doubly-indirect: addrs[NDIRECT+1]
    if (bn < NINDIRECT * NINDIRECT) {
        uint idx1 = bn / NINDIRECT;   // first-level index
        uint idx2 = bn % NINDIRECT;   // second-level index

        // Load (or allocate) doubly-indirect root block
        if ((addr = ip->addrs[NDIRECT+1]) == 0) {
            addr = balloc(ip->dev);
            ip->addrs[NDIRECT+1] = addr;
        }
        bp = bread(ip->dev, addr);
        a = (uint *)bp->data;

        // Load (or allocate) first-level indirect block
        if ((addr = a[idx1]) == 0) {
            addr = balloc(ip->dev);
            a[idx1] = addr;
            log_write(bp);
        }
        brelse(bp);

        // Load (or allocate) data block via second-level
        bp = bread(ip->dev, addr);
        a = (uint *)bp->data;
        if ((addr = a[idx2]) == 0) {
            addr = balloc(ip->dev);
            a[idx2] = addr;
            log_write(bp);
        }
        brelse(bp);
        return addr;
    }

    panic("bmap: out of range");
}
```

### 3. itrunc() — freeing doubly-indirect blocks

```c
void itrunc(struct inode *ip) {
    // ... free direct blocks ...
    // ... free singly-indirect block ...

    // Free doubly-indirect block
    if (ip->addrs[NDIRECT+1]) {
        struct buf *bp = bread(ip->dev, ip->addrs[NDIRECT+1]);
        uint *a = (uint *)bp->data;

        for (int j = 0; j < NINDIRECT; j++) {
            if (a[j]) {
                struct buf *bp2 = bread(ip->dev, a[j]);
                uint *a2 = (uint *)bp2->data;

                // Free all data blocks in this second-level block
                for (int k = 0; k < NINDIRECT; k++) {
                    if (a2[k])
                        bfree(ip->dev, a2[k]);
                }
                brelse(bp2);
                bfree(ip->dev, a[j]);   // free the second-level indirect block
            }
        }
        brelse(bp);
        bfree(ip->dev, ip->addrs[NDIRECT+1]);  // free the root doubly-indirect block
        ip->addrs[NDIRECT+1] = 0;
    }

    ip->size = 0;
    iupdate(ip);
}
```

### 4. sys_symlink()

```c
uint64 sys_symlink(void) {
    char target[MAXPATH], path[MAXPATH];

    if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
        return -1;

    begin_op();

    // Create a new inode of type T_SYMLINK
    struct inode *ip = create(path, T_SYMLINK, 0, 0);
    if (ip == 0) { end_op(); return -1; }

    // Store target path as the symlink's file content
    if (writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)) {
        iunlockput(ip);
        end_op();
        return -1;
    }

    iunlockput(ip);
    end_op();
    return 0;
}
```

### 5. sys_open() — following symlinks

```c
// After resolving the initial path and locking the inode:
if (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    int depth = 0;
    char target[MAXPATH];

    while (ip->type == T_SYMLINK && depth < 10) {
        // Read target path from symlink content
        if (readi(ip, 0, (uint64)target, 0, MAXPATH) <= 0) {
            iunlockput(ip);
            end_op();
            return -1;
        }
        iunlockput(ip);         // release before namei

        ip = namei(target);     // resolve new path (no lock held)
        if (ip == 0) { end_op(); return -1; }

        ilock(ip);              // re-lock before reading ip->type
        depth++;
    }

    // If still a symlink after max depth: cycle detected
    if (depth >= 10 && ip->type == T_SYMLINK) {
        iunlockput(ip);
        end_op();
        return -1;
    }
}
// Continue with normal open logic...
```

---

## Design Decisions

### 1. Trade One Direct Block for Doubly-Indirect

On-disk inode size is fixed, so we can't just append another address slot without breaking compatibility.

A practical way is a size-neutral swap:
- Remove one direct block slot (NDIRECT: 12 → 11)
- Add one doubly-indirect slot in its place
- Array size stays at 13 entries (`NDIRECT+2 = 13`, same bytes as before)

Trade-off: one fewer direct slot for tiny files, in exchange for much larger max file size.

### 2. bmap() Allocates on Demand

`bmap()` is not only a lookup routine; it also allocates blocks on demand. So it must run inside a transaction (`begin_op`/`end_op`) and log metadata changes with `log_write()`.

The important sequencing in the doubly-indirect case:
1. Allocate root block if missing → `log_write` parent inode
2. Allocate first-level block if missing → `log_write` root block
3. Allocate data block if missing → `log_write` first-level block

If you skip logging at any level, a crash can leave partially linked metadata.

### 3. Symlinks Store Plain Path Strings

Symlink target is stored as a plain path string in file data (no special encoding or extra metadata).

Simple and sufficient for xv6. The known downside is dangling symlinks when target paths change.

### 4. inode Lock Protocol for Symlink Following

Don't call `namei()` while holding an inode lock — `namei()` acquires locks internally and can deadlock.

The required sequence for each hop:
```
read ip->type    (ilock held)
read target path (ilock held)
iunlockput(ip)   (release before namei)
ip = namei(...)  (no lock held)
ilock(ip)        (lock new inode)
```

If this order is broken, deadlocks are very possible.

### 5. Cycle Detection via Depth Limit

Exact cycle detection (A → B → A) needs visited-set tracking. xv6 keeps it simple with a depth limit of 10.

If depth reaches 10 and the current inode is still a symlink, we assume a cycle and return an error. This matches Linux's historical approach (`MAXSYMLINKS = 8` on older kernels, now 40).

---

## Common Pitfalls

### 1. Forgetting to Subtract Previous Ranges in bmap()

Each indirect tier must subtract the block count of all previous tiers before computing its local index.

Missing the subtraction:
```c
// Doubly-indirect case
uint idx1 = bn / NINDIRECT;   // bn was never adjusted for singly-indirect range!
```

With the subtraction:
```c
bn -= NDIRECT;
if (bn < NINDIRECT) { /* singly-indirect */ }
bn -= NINDIRECT;              // subtract singly-indirect range
// now bn is relative to the doubly-indirect range
uint idx1 = bn / NINDIRECT;
```

### 2. Freeing Blocks in the Wrong Order in itrunc()

Freeing the root block first, then trying to read child pointers from it:
```c
bfree(ip->dev, ip->addrs[NDIRECT+1]);  // root freed!
bp = bread(ip->dev, ip->addrs[NDIRECT+1]);  // reading freed block — garbage!
```

Always free leaves before their parents.

### 3. Not Calling log_write() After Modifying Indirect Blocks

`bmap()` modifies disk block content (the pointer arrays inside indirect blocks). These changes must be logged:

```c
a[idx1] = addr;
log_write(bp);   // must come immediately after modifying bp->data
```

Without `log_write`, the change exists only in the buffer cache. A crash before it's written to disk leaves a zero entry, causing data loss.

### 4. Calling namei() While Holding an Inode Lock

```c
// Wrong: namei acquires locks internally, can deadlock
ilock(ip);
// ... read symlink target ...
ip2 = namei(target);   // deadlock if namei tries to lock a parent of ip!
```

```c
// Correct: release before resolving
iunlockput(ip);
ip2 = namei(target);
ilock(ip2);
```

### 5. Not Handling the O_NOFOLLOW Flag

When `O_NOFOLLOW` is set, `open()` must return the symlink inode itself — not the target. Forgetting this check means symlinks can never be opened directly (you can't read their content, remove them, etc. with standard tools).

```c
if (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    // follow the link
}
// If O_NOFOLLOW is set, fall through to normal open
```

### 6. Infinite Loop Without a Depth Limit

Without a maximum depth, `A → B → A` causes an infinite loop in `sys_open()`, hanging the process forever (or until stack overflow).

Check depth on each hop and fail once the limit is reached.

---

## How the Pieces Fit Together

```
write(fd, buf, n)
  → writei(ip, buf, offset, n)
      → bmap(ip, block_number)        ← translates logical → disk block
          if direct: ip->addrs[bn]
          if singly-indirect: bread(addrs[NDIRECT]) → a[bn]
          if doubly-indirect: bread(addrs[NDIRECT+1])
                               → a[idx1] → bread(a[idx1]) → a[idx2]
      → bread / bwrite the data block

open("link", 0)
  → namei("link") → inode 42 (T_SYMLINK)
  → ilock(42), read target="/real/file"
  → iunlockput(42)
  → namei("/real/file") → inode 17
  → ilock(17), continue normal open
```

`bmap()` is the core translation layer in this lab. Once this function is correct, the rest of the file growth path is much easier to reason about.
