# xv6 Traps

https://pdos.csail.mit.edu/6.1810/2025/labs/traps.html

## Table of Contents
1. [What is a Trap?](#what-is-a-trap)
2. [RISC-V Trap Mechanism](#risc-v-trap-mechanism)
3. [Lab Tasks Overview](#lab-tasks-overview)
4. [Task 1: Backtrace](#task-1-backtrace)
5. [Task 2: Alarm](#task-2-alarm)
6. [Pseudo Code](#pseudo-code)
7. [Design Decisions](#design-decisions)
8. [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)

---

## What is a Trap?

A **trap** is a transfer of control from user space to kernel space. In RISC-V (and xv6), three events cause a trap:

| Event | Cause | Example |
|-------|-------|---------|
| **System call** | `ecall` instruction | `write()`, `fork()` |
| **Exception** | Illegal instruction or memory access | Page fault, divide by zero |
| **Device interrupt** | Hardware signals the CPU | Timer tick, disk I/O done |

All three go through the **same trap handling path** — the hardware saves minimal state, jumps to the kernel's trap handler, the kernel figures out what happened, and eventually returns to user space.

---

## RISC-V Trap Mechanism

### Hardware's Role (What the CPU does automatically)

When a trap occurs, RISC-V hardware:
1. Saves the current PC into `sepc` (supervisor exception PC)
2. Saves the cause into `scause`
3. Sets privilege mode to supervisor
4. Jumps to the address in `stvec` (supervisor trap vector)

The hardware saves very little. Registers, stack pointer, everything else — the kernel must save them manually. This is why xv6 has a dedicated **trapframe** per process.

### The Trapframe

The trapframe is a per-process kernel structure that holds a snapshot of all 32 user registers at the moment a trap occurs:

```
struct trapframe {
    uint64 kernel_satp;   // kernel page table
    uint64 kernel_sp;     // kernel stack pointer
    uint64 kernel_trap;   // address of usertrap()
    uint64 epc;           // saved user program counter  ← sepc
    uint64 kernel_hartid; // CPU core ID
    uint64 ra, sp, gp, tp;
    uint64 t0 ... t6;
    uint64 s0 ... s11;
    uint64 a0 ... a7;     // a0-a7: syscall args / return value
}
```

To **return to user space**, the kernel:
1. Restores all registers from the trapframe
2. Sets `sepc` to `trapframe->epc` (where user code should resume)
3. Executes `sret` (supervisor return)

### RISC-V Stack Frame Layout

Each function call creates a **stack frame** on the stack:

```
High address
+------------------+
|  return address  |  ← fp - 8   (where to return after this function)
+------------------+
|  saved fp (s0)   |  ← fp - 16  (previous frame's fp)
+------------------+
|  local variables |
|  ...             |
+------------------+  ← sp (stack pointer, grows downward)
Low address
```

The **frame pointer (s0/fp)** always points to the top of the current frame. By following the chain of saved frame pointers, you can walk up the entire call stack — this is what `backtrace()` does.

---

## Lab Tasks Overview

This lab has two independent tasks:

1. Backtrace — Walk the kernel call stack and print return addresses (debugging utility)
2. Alarm — Implement `sigalarm(interval, handler)` / `sigreturn()` to deliver periodic software interrupts to user processes

---

## Task 1: Backtrace

### Goal

Implement `backtrace()` in the kernel that prints the return address of every stack frame on the current kernel stack. Called automatically in `panic()`.

### How Stack Walking Works

```
Current frame:
  fp → [return addr | saved fp | locals...]
           ↓              ↓
        print this    jump here → repeat
```

Starting from the current `s0` register (the frame pointer), we:
1. Read the return address at `fp - 8`, print it
2. Load the previous frame pointer from `fp - 16`
3. Repeat until we leave the current page (xv6 allocates exactly one page per kernel stack)

### When to Stop

xv6 gives each kernel stack exactly **one page**. When the frame pointer moves outside this page boundary, we've walked off the top of the stack.

---

## Task 2: Alarm

### Goal

Implement two new system calls:

- `sigalarm(n, handler)` — every `n` CPU ticks, call `handler()` in user space
- `sigreturn()` — called by the handler to resume the interrupted code

### The Challenge: Context Preservation

The tricky part is that `handler()` runs **in the middle of** whatever the user was doing. After the handler returns (via `sigreturn`), the user program must continue as if **nothing happened** — the same registers, the same PC, the same return values.

This requires saving and restoring the **entire trapframe**.

### Flow Overview

```
User program running normally
  → timer interrupt (every N ticks)
  → kernel increments alarm_ticks
  → alarm_ticks == alarm_interval?
      → save full trapframe to alarm_trapframe
      → set trapframe->epc = alarm_handler address
      → return to user space (but now running the handler!)
  → handler does its work
  → handler calls sigreturn()
      → restore trapframe from alarm_trapframe
      → clear in-handler flag
      → return to user space (resuming original code)
```

### Re-entrancy Prevention

What if the handler takes longer than `n` ticks? Without a guard, the timer would interrupt the handler and try to call it again — recursively, until stack overflow.

Solution: an `alarm_in_handler` flag. If it's set, skip the alarm even if the tick count fires.

---

## Code Sketches

These snippets are intentionally simplified. They match xv6 behavior but are not copy-paste C code.

### 1. Reading the Frame Pointer

```c
// Read the s0 register (frame pointer) via inline assembly
static inline uint64 r_fp() {
    uint64 x;
    asm volatile("mv %0, s0" : "=r"(x));
    return x;
}
```

RISC-V calling convention keeps the current frame pointer in `s0`, so reading `s0` is the stable entry point for stack walking.

### 2. backtrace()

```c
void backtrace(void) {
    uint64 fp = r_fp();
    uint64 page_start = PGROUNDDOWN(fp);

    printf("backtrace:\n");

    while (fp >= page_start && fp < page_start + PGSIZE) {
        uint64 ra = *(uint64 *)(fp - 8);   // return address
        printf("%p\n", (void *)ra);
        fp = *(uint64 *)(fp - 16);          // previous frame pointer
    }
}
```

**Why one page?** xv6 allocates exactly `PGSIZE` bytes for each kernel stack. Once `fp` leaves this range, we've walked past the bottom of the stack.

### 3. Process Structure (Alarm Fields)

```c
struct proc {
    // ... existing fields ...
    int alarm_interval;               // N ticks between calls
    void (*alarm_handler)();          // user-space handler address
    int alarm_ticks;                  // elapsed ticks since last alarm
    struct trapframe alarm_trapframe; // saved snapshot of registers
    int alarm_in_handler;             // re-entrancy guard
};
```

### 4. sys_sigalarm()

```c
uint64 sys_sigalarm(void) {
    int interval;
    uint64 handler;
    struct proc *p = myproc();

    argint(0, &interval);
    argaddr(1, &handler);

    p->alarm_interval = interval;
    p->alarm_handler  = (void(*)())handler;
    p->alarm_ticks    = 0;
    return 0;
}
```

### 5. Timer Interrupt in usertrap()

```c
// inside usertrap(), after detecting which_dev == 2 (timer)
if (p->alarm_interval > 0 && !p->alarm_in_handler) {
    p->alarm_ticks++;
    if (p->alarm_ticks >= p->alarm_interval) {
        p->alarm_ticks = 0;
        // Save full register state
        memmove(&p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
        // Prevent re-entrant calls
        p->alarm_in_handler = 1;
        // Redirect return to handler — not a direct call!
        p->trapframe->epc = (uint64)p->alarm_handler;
    }
}
yield();
```

The important detail: we do not directly call the handler from kernel code. We only rewrite `trapframe->epc`. After `sret`, user mode resumes at the handler address.

### 6. sys_sigreturn()

```c
uint64 sys_sigreturn(void) {
    struct proc *p = myproc();

    // Restore the full register snapshot taken before the handler
    memmove(p->trapframe, &p->alarm_trapframe, sizeof(struct trapframe));

    // Clear re-entrancy guard
    p->alarm_in_handler = 0;

    // Return saved a0 so the syscall machinery writes it back to trapframe->a0
    // net effect: trapframe->a0 = alarm_trapframe.a0 — no clobbering
    return p->alarm_trapframe.a0;
}
```

---

## Design Decisions

### 1. Save the Entire Trapframe (Not Just epc)

The obvious first attempt is to save only `epc`. This breaks because handler code is normal C and freely clobbers registers (`a*`, `t*`, etc). The right approach is to save the entire trapframe into `alarm_trapframe` and restore it in `sigreturn`.

```
Before alarm fires:       After sigreturn:
  a0 = 42                   a0 = 42      ← preserved
  t1 = 0xdeadbeef           t1 = 0xdeadbeef  ← preserved
  epc = 0x1234              epc = 0x1234 ← resumes here
```

### 2. Redirecting via epc Instead of Direct Call

The kernel cannot directly call a user-space function pointer. Kernel and user space have different address spaces and privilege levels. Instead, we **manipulate the trap return path**:

- Change `trapframe->epc` to the handler address
- When `sret` executes, the CPU jumps to that address in user mode

In practice this means we reuse the normal trap return path, but with a different resume PC.

### 3. `a0` Return-Value Clobbering in `sigreturn`

`sigreturn` is a system call. The syscall machinery in xv6 always writes the return value of the syscall into `trapframe->a0`:

```
trapframe->a0 = syscall_return_value
```

If `sigreturn` returns `0`, syscall glue overwrites restored `a0` with `0`, which breaks user-state restoration.

**Solution:** Return the saved `a0` from `sys_sigreturn()`. The syscall machinery then writes this same value back, resulting in a no-op:

```c
return p->alarm_trapframe.a0;  // syscall will write this to trapframe->a0
                                // net effect: trapframe->a0 = alarm_trapframe.a0 ✓
```

### 4. Re-entrancy Guard (`alarm_in_handler`)

| Scenario | Without Guard | With Guard |
|----------|--------------|------------|
| Handler takes < N ticks | Fine | Fine |
| Handler takes > N ticks | Alarm fires again inside handler | Skipped until handler returns |
| Handler calls sigreturn mid-way | Undefined | Re-enabled at sigreturn |

The guard (`alarm_in_handler`) is set when the alarm fires and cleared in `sigreturn` — not when the handler function returns, since handler → sigreturn is the only correct exit path.

### 5. When to Reset `alarm_ticks`

Reset `alarm_ticks = 0` **when alarm fires**, not in `sigreturn`. So the next interval starts at fire time, not handler-finish time.

---

## Common Pitfalls

### 1. Not Saving All Registers Before the Handler

Only saving the PC:
```c
p->alarm_trapframe.epc = p->trapframe->epc;  // only save PC
p->trapframe->epc = (uint64)p->alarm_handler;
```

Save the full trapframe instead:
```c
memmove(&p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));  // save everything
p->trapframe->epc = (uint64)p->alarm_handler;
```

Any register the handler touches — including compiler-generated temporaries — will be wrong when user code resumes.

### 2. Forgetting the a0 Return Value Issue in sigreturn

Returning 0 here lets the syscall machinery overwrite the restored `a0`:
```c
uint64 sys_sigreturn(void) {
    memmove(p->trapframe, &p->alarm_trapframe, sizeof(struct trapframe));
    p->alarm_in_handler = 0;
    return 0;  // syscall machinery writes 0 to trapframe->a0 — corrupts a0!
}
```

Return the saved `a0` instead so the write-back is a no-op:
```c
uint64 sys_sigreturn(void) {
    memmove(p->trapframe, &p->alarm_trapframe, sizeof(struct trapframe));
    p->alarm_in_handler = 0;
    return p->alarm_trapframe.a0;  // preserved correctly
}
```

### 3. Re-entrancy: Set Guard Before Redirecting

Set the guard first, then redirect:
```c
// wrong order:
p->trapframe->epc = (uint64)p->alarm_handler;  // redirect first
p->alarm_in_handler = 1;                        // set guard after

// correct order:
p->alarm_in_handler = 1;                        // guard first
p->trapframe->epc = (uint64)p->alarm_handler;
```

On real systems this ordering matters. In xv6 it is less likely to bite, but the safer order is still worth keeping.

### 4. Stopping Backtrace at the Wrong Boundary

Stopping at `fp != 0` can walk into unrelated memory. Use the page boundary instead:
```c
// wrong:
while (fp != 0) { ... }

// correct:
uint64 page_start = PGROUNDDOWN(fp);
while (fp >= page_start && fp < page_start + PGSIZE) {
    ...
}
```

xv6 kernel stacks are one page each. The page boundary check is both sufficient and safe.

### 5. Wrong Offsets for Stack Frame Fields

RISC-V ABI defines the frame layout precisely:
- Return address: `fp - 8`
- Saved frame pointer: `fp - 16`

These are fixed by the calling convention — do not guess or change them.

