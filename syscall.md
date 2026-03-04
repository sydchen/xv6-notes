# xv6 System Calls

https://pdos.csail.mit.edu/6.1810/2025/labs/syscall.html

## Table of Contents
1. [How System Calls Work](#how-system-calls-work)
2. [Lab Tasks Overview](#lab-tasks-overview)
3. [Task 1: GDB Debugging](#task-1-gdb-debugging)
4. [Task 2: interpose() — Process Sandboxing](#task-2-interpose--process-sandboxing)
5. [Task 3: Memory Attack](#task-3-memory-attack)
6. [Code Sketches](#code-sketches)
7. [Key Design Decisions](#key-design-decisions)

---

## How System Calls Work

A system call is a controlled jump from user space into the kernel. The full path in xv6:

```
user code calls write()
  → usys.S stub: loads syscall number into a7, executes ecall
  → hardware: saves PC to sepc, switches to supervisor mode, jumps to stvec
  → uservec (trampoline): saves all registers into trapframe
  → usertrap(): identifies cause as syscall (scause == 8)
  → syscall(): reads a7, dispatches to syscalls[num]()
  → sys_write(): reads args from trapframe a0–a5, does work
  → returns value → written to trapframe->a0
  → userret: restores registers, sret
  → user code continues, a0 holds return value
```

### Syscall Number Dispatch

`syscall.c` maintains a simple jump table:

```
syscalls[SYS_write]  = sys_write
syscalls[SYS_read]   = sys_read
syscalls[SYS_fork]   = sys_fork
...
```

`a7` contains the syscall number (defined in `syscall.h`). The dispatcher looks up the function pointer and calls it. The return value is stored into `trapframe->a0`, which the user sees as the function's return value.

### How Arguments Are Passed

User space puts arguments in registers `a0`–`a5`. The kernel reads them from the **trapframe** (not from registers directly, since it's in a different context):

```
argint(n, &val)    → reads trapframe->a0 + n*8 as integer
argaddr(n, &val)   → reads trapframe->a0 + n*8 as address
argstr(n, buf, max) → reads pointer from trapframe, then copies string from user space
```

`argstr` is the tricky one: it reads a *pointer* from the trapframe, then must copy the string **from user address space** into kernel memory — which requires `copyinstr()`, not a simple dereference. Directly dereferencing a user pointer in the kernel is a security bug.

---

## Lab Tasks Overview

| Task | What you build | Key concept |
|------|---------------|-------------|
| GDB debugging | Answer questions about trap state | Trapframe layout, sepc |
| `interpose()` | Syscall sandboxing via bitmask | Bitmask enforcement, fork inheritance |
| Memory attack | Recover secret from freed pages | Kernel memory zeroing (or lack thereof) |

---

## Task 1: GDB Debugging

### What to Look At

Setting a breakpoint at `syscall()` and inspecting state teaches you:

- `p->trapframe->a7` holds the syscall number
- `p->trapframe->epc` holds the address of the `ecall` instruction (sepc)
- After `syscall()` returns, `epc` is incremented by 4 so user code resumes at the **next** instruction

### Typical GDB Session

```
(gdb) break syscall
(gdb) continue
(gdb) print p->trapframe->a7     # syscall number
(gdb) print/x p->trapframe->epc  # address of ecall
(gdb) x/i p->trapframe->epc      # disassemble: should show ecall
```

### Kernel Panic Backtrace

When the kernel panics, it calls `backtrace()` (implemented in the Traps lab). The addresses printed map to kernel functions — use `addr2line` or `nm kernel/kernel` to decode them.

---

## Task 2: interpose() — Process Sandboxing

### Goal

Add a system call `interpose(mask, allowed_path)` that restricts what syscalls the calling process can make:

- `mask`: bitmask of **forbidden** syscall numbers (bit `n` set → syscall `n` is blocked)
- `allowed_path`: a special exception for `open` and `exec` — if a forbidden call targets this path, allow it anyway

### Data Model

Two fields added to `struct proc`:

```
struct proc {
    ...
    int  syscall_mask;       // bitmask: bit n set = syscall n forbidden
    char allowed_path[128];  // exception path for open/exec
}
```

### Enforcement Point

The check happens in `syscall()`, just before dispatch:

```c
if ((p->syscall_mask >> num) & 1) {
    if (num == SYS_open || num == SYS_exec) {
        // handled inside the syscall itself (needs path arg)
        p->trapframe->a0 = syscalls[num]();
    } else {
        p->trapframe->a0 = -1;  // blocked
    }
} else {
    p->trapframe->a0 = syscalls[num]();
}
```

`open` and `exec` are special because the enforcement depends on **which file** is being opened/executed — a piece of information only available inside the syscall handler, not at the dispatch layer.

### Fork Inheritance

The sandbox must be inherited across `fork()`. A child process should be at least as restricted as its parent:

```c
// inside fork()
np->syscall_mask = p->syscall_mask;
safestrcpy(np->allowed_path, p->allowed_path, sizeof(np->allowed_path));
```

Without inheritance, a sandboxed process could simply `fork()` and have the child make unrestricted syscalls.

---

## Task 3: Memory Attack

### The Vulnerability

xv6's `kalloc()` normally zeroes newly allocated pages (`memset`). If that zeroing is **removed** (as the lab does intentionally), freed pages retain their old contents. When a new process calls `sbrk()` and receives one of these pages, it can read whatever was stored there before.

### The Attack Strategy

A "secret" process allocates a buffer, writes sensitive data, then exits — returning the pages to the free list. The attacker process then:

1. Calls `sbrk()` to allocate a large chunk of memory
2. Scans the received pages for recognizable patterns (a known marker string)
3. Reads the secret data adjacent to the marker

```
Memory layout of victim's buffer:
  offset 0:  "This may help."   ← known marker
  offset 16: <secret 8 bytes>   ← target
```

### Why This Works

The kernel free list is LIFO (last in, first out — a stack). When the victim exits and frees pages, those pages go to the top of the free list. The attacker's `sbrk()` pulls pages from the same list. With no zeroing, the old content is intact.

### Lesson

This is a real class of vulnerability: **use-after-free information disclosure**. The fix is always zeroing on allocation, not on free. Zeroing on free is insufficient because the content already leaked to whoever allocated it next.

---

## Code Sketches

### 1. sys_interpose()

```c
uint64
sys_interpose(void)
{
    struct proc *p = myproc();
    int mask;
    uint64 path_addr;

    argint(0, &mask);
    argaddr(1, &path_addr);

    p->syscall_mask = mask;
    argstr(1, p->allowed_path, sizeof(p->allowed_path));
    return 0;
}
```

### 2. Enforcement in syscall()

```c
void
syscall(void)
{
    struct proc *p = myproc();
    int num = p->trapframe->a7;

    if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        int blocked = (p->syscall_mask >> num) & 1;
        int path_syscall = (num == SYS_open || num == SYS_exec);

        if (blocked && !path_syscall) {
            p->trapframe->a0 = -1;      // deny immediately
        } else {
            p->trapframe->a0 = syscalls[num]();  // call (may self-check path)
        }
    } else {
        printf("unknown syscall %d\n", num);
        p->trapframe->a0 = -1;
    }
}
```

### 3. Path-based Check Inside sys_open() / sys_exec()

```c
uint64
sys_open(void)
{
    struct proc *p = myproc();
    char path[MAXPATH];
    int flags;

    argstr(0, path, MAXPATH);
    argint(1, &flags);

    if ((p->syscall_mask >> SYS_open) & 1) {
        if (strncmp(p->allowed_path, "-", 2) != 0 &&
            strncmp(path, p->allowed_path, MAXPATH) == 0) {
            // exception: this path is explicitly allowed
        } else {
            return -1;
        }
    }

    // proceed with normal open logic
    ...
}
```

### 4. fork() Inheritance

```c
int
fork(void)
{
    struct proc *p = myproc();
    struct proc *np;

    // ... copy memory, file descriptors, etc. ...

    np->syscall_mask = p->syscall_mask;
    safestrcpy(np->allowed_path, p->allowed_path, sizeof(np->allowed_path));

    ...
}
```

### 5. Memory Attack

```c
void
attack(void)
{
    // Allocate pages — may receive victim's freed pages
    char *buf = sbrk(LARGE_SIZE);

    for (int i = 0; i < LARGE_SIZE - 24; i++) {
        if (strncmp(&buf[i], "This may help.", 14) == 0) {
            char *secret = &buf[i + 16];  // 8-byte secret
            write(1, secret, 8);
            exit(0);
        }
    }

    printf("not found\n");
    exit(1);
}
```

---

## Key Design Decisions

### 1. Bitmask Representation

A bitmask is the natural choice for a set of syscall numbers:

```
mask = (1 << SYS_write) | (1 << SYS_open)
```

- O(1) check: `(mask >> num) & 1`
- Fits in a single `int` (xv6 has ~25 syscalls)
- Easy to combine: `child_mask = parent_mask | extra_restrictions`

I considered a list/array of blocked syscall numbers, but the bitmask check is simpler and cheaper here.

### 2. Why open and exec Are Special

Most syscalls can be blocked at the dispatch layer without knowing their arguments. But `open` and `exec` have a path-based exception: even if they're "forbidden," a specific file path is still allowed.

So enforcement **has to happen inside the syscall**, after path arguments are read from user space. Doing this at dispatch time would duplicate parsing and is easy to break.

This keeps dispatch logic simple and leaves path-aware checks to handlers that already parse paths.

### 3. allowed_path = "-" Means No Exception

`"-"` means "no path exception; block all opens/execs." It is a bit opaque, but acceptable for xv6. A cleaner production design would use a dedicated boolean flag.

### 4. Zeroing on Allocation, Not on Free

| Strategy | When zero | Leak risk |
|----------|-----------|-----------|
| Zero on free | At free time | Allocator can still hand out non-zero pages from a crash path |
| Zero on alloc | At alloc time | Recipient always gets zeroed page — no prior content |

The key invariant is: every `kalloc()` caller gets a clean page. Zeroing only on free can still leak stale data.

