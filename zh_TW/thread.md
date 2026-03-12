# xv6 Threads

https://pdos.csail.mit.edu/6.1810/2023/labs/thread.html

## 目錄
1. [背景：什麼是 Thread？](#背景什麼是-thread)
2. [Lab 任務總覽](#lab-任務總覽)
3. [任務 1：使用者層執行緒切換（uthread）](#任務-1使用者層執行緒切換uthread)
4. [任務 2：加鎖的 Hash Table](#任務-2加鎖的-hash-table)
5. [任務 3：使用 Condition Variable 的 Barrier](#任務-3使用-condition-variable-的-barrier)
6. [Code Sketches](#code-sketches)
7. [設計決策](#設計決策)
8. [常見陷阱](#常見陷阱)

---

## 背景：什麼是 Thread？

thread 是可獨立執行、且與同一行程內其他 thread 共享記憶體的執行上下文。每個 thread 需要自己的：
- Stack：區域變數與函式呼叫框架
- Register state：PC、SP 與保存的暫存器

從 thread A 切到 thread B 的本質是：
1. 把 A 的暫存器存起來
2. 還原 B 的暫存器
3. 跳到 B 保存的 PC（透過 `ra` + `ret`）

其他像 heap、global、file descriptor 都是共享的。

### Cooperative vs Preemptive

| 模型 | 何時切換？ | 需要 |
|------|------------|------|
| Cooperative | thread 自己呼叫 `yield()` | 額外支援少 |
| Preemptive | timer interrupt 強制切換 | kernel 支援 |

這個 lab 的 uthread 是 cooperative：在 user space 切換，不靠 kernel preemption。

---

## Lab 任務總覽

| Task | 環境 | 實作內容 |
|------|------|----------|
| uthread | xv6 user space | `thread_create()`, `thread_schedule()`, `thread_switch.S` |
| Hash table | Linux/macOS POSIX | 加 mutex 消除 data race |
| Barrier | Linux/macOS POSIX | 用 condition variable + 世代 的 barrier |

任務 2、3 跑在 POSIX pthread 環境，不在 xv6 內。

---

## 任務 1：使用者層執行緒切換（uthread）

### Data Structures

```c
struct context {
    uint64 ra;          // return address — resume 位置
    uint64 sp;          // stack pointer
    uint64 s0 ... s11;  // callee-saved registers
};

struct thread {
    char           stack[STACK_SIZE];  // 8192 bytes, 內嵌在 struct
    int            state;              // FREE / RUNNABLE / RUNNING
    struct context context;            // 保存的暫存器狀態
};
```

只需要保存 callee-saved（`ra`, `sp`, `s0`–`s11`）。caller-saved（`a0`–`a7`, `t0`–`t6`）本來就不保證跨函式呼叫保留，`thread_yield()` 前後也不該假設它們不變。

### thread_create()

讓新 thread 能被第一次排程時正確啟動：

```c
void thread_create(void (*func)()) {
    // 找 FREE slot
    struct thread *t = find_free_thread();
    t->state = RUNNABLE;

    // stack top（stack 向下成長）
    t->context.sp = (uint64)(t->stack + STACK_SIZE);

    // thread_switch 還原 ra 後 ret 會跳到這裡
    t->context.ra = (uint64)func;

    // 其他暫存器初始為 0
}
```

`thread_switch` 最後是 `ret`，而 `ret` 跳到 `ra`。把 `ra = func`，第一次被排到就會直接進 `func()`。

### thread_schedule()

```c
void thread_schedule(void) {
    // round-robin 找下一個 RUNNABLE
    struct thread *next = find_next_runnable();

    if (current_thread != next) {
        next->state = RUNNING;
        struct thread *old = current_thread;
        current_thread = next;
        // save old, restore next, jump to next->context.ra
        thread_switch((uint64)&old->context, (uint64)&next->context);
    }
}
```

### thread_switch.S — 核心組語

`thread_switch(a0, a1)`：
- `a0` = old thread 的 `struct context` 指標（保存到這裡）
- `a1` = new thread 的 `struct context` 指標（從這裡還原）

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

    ret   # 跳到還原後的 ra
```

`ret` 之後，CPU 已在新 thread 的 stack 上，從新 thread 的 `ra` 繼續執行。從新 thread 視角看，像是 `thread_switch()` 剛返回。

### Thread Lifecycle

```text
thread_create(func)
  → state = RUNNABLE
  → context.ra = func, context.sp = stack top

thread_schedule() 選到這個 thread
  → thread_switch() 還原 context
  → ret → 跳進 func()

func() 執行，呼叫 thread_yield()
  → state = RUNNABLE
  → thread_schedule() → 切走

func() 執行完
  → state = FREE
  → thread_schedule() → 不再被選到
```

---

## 任務 2：加鎖的 Hash Table

### Race Condition

hash table 用 separate chaining（每個 bucket 是 linked list）。`put(key, value)` 會：
1. 算 `bucket = key % NBUCKET`
2. 走訪 bucket list 找 key
3. 找不到就把新節點插到 head

兩個 thread 同時在同一 bucket 做第 3 步時：

```text
Thread 1: new_node->next = bucket[b]   // 看到舊 head
Thread 2: new_node->next = bucket[b]   // 也看到舊 head
Thread 1: bucket[b] = new_node1        // 安裝 node1
Thread 2: bucket[b] = new_node2        // 覆蓋 node1
```

結果會丟失一筆 `put()`。

### Coarse vs Fine-Grained Locking

**Coarse（全域單鎖）**：正確但慢，所有 `put/get` 串行。

```c
pthread_mutex_t global_lock;

void put(int key, int value) {
    pthread_mutex_lock(&global_lock);
    // ... insert ...
    pthread_mutex_unlock(&global_lock);
}
```

**Fine-grained（每個 bucket 一把鎖）**：正確且可並行。

```c
pthread_mutex_t locks[NBUCKET];

void put(int key, int value) {
    int b = key % NBUCKET;
    pthread_mutex_lock(&locks[b]);
    // ... insert into bucket b ...
    pthread_mutex_unlock(&locks[b]);
}
```

每 bucket 一把鎖時，不同 bucket 的操作可以平行。

---

## 任務 3：使用 Condition Variable 的 Barrier

### Barrier 做什麼

barrier 保證 N 個 thread 都到達同步點前，沒有任何 thread 可以先往下跑：

```text
Thread 1 ───────────────────►|
Thread 2 ──────►|            |  (waits here)
Thread 3 ──────────────────►|
                              ▼ all continue
```

### 實作

```c
struct barrier {
    pthread_mutex_t    lock;
    pthread_cond_t     cond;
    int                count;   // 目前等待中的 thread 數
    int                round;   // 目前 barrier 世代
    int                nthread; // thread 總數
};

void barrier() {
    pthread_mutex_lock(&b.lock);

    b.count++;
    if (b.count == b.nthread) {
        // 最後一個到達者：喚醒全部
        b.round++;
        b.count = 0;
        pthread_cond_broadcast(&b.cond);
    } else {
        // 要防 spurious wakeup 與跨輪問題
        int my_round = b.round;
        while (b.round == my_round)
            pthread_cond_wait(&b.cond, &b.lock);
    }

    pthread_mutex_unlock(&b.lock);
}
```

### 為什麼要追蹤 `round`？

若沒有 `round`，快的 thread 可能在慢 thread 還沒離開第一輪 barrier 前就跑回來進入下一輪，導致世代混在一起。

`round` 讓每個 thread 帶著 世代 概念等待正確那一輪的 broadcast：

```text
my_round = b.round
while (b.round == my_round)
    pthread_cond_wait(...)
// b.round 改變才表示該輪真的放行
```

---

## Code Sketches

### 1. struct context / struct thread

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

    t->context.sp = (uint64)(t->stack + STACK_SIZE); // stack top
    t->context.ra = (uint64)func;                    // first PC on ret
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

// get — put 可能並行時也要加鎖
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

## 設計決策

### 1. 只保存 Callee-Saved Registers

C calling convention 保證 caller-saved（`a0`–`a7`, `t0`–`t6`）本來就不會跨函式保留。會呼叫 `thread_yield()` 的程式也不該依賴它們不變。

所以 context switch 只要保存 `ra`, `sp`, `s0`–`s11`。全存 32 個暫存器雖可行，但成本更高。

### 2. 用 `ra` 當首次進入 PC

新 thread 第一次跑時，沒有既有執行狀態可還原，只有 `thread_create` 預先填好的 context。

把 `context.ra = func`，配合 `thread_switch` 的 `ret`，就能第一次排程直接進 `func`。`context.sp` 設在 stack top，確保 `func` 從第一條指令就有合法 stack。

### 3. Stack 內嵌在 struct thread

stack 直接放在 `struct thread`（固定大小 char 陣列）裡，配置簡單：在 `all_thread[]` 找空位即可，不用另外 `malloc`。

代價是沒有 guard page，overflow 可能破壞鄰近 thread struct。對教學版實作可接受。

### 4. 每 Bucket 一把鎖

| 鎖策略 | 正確性 | 效能 |
|--------|--------|------|
| 不加鎖 | 錯（會丟 key） | 快 |
| 全域單鎖 | 對 | 差（無平行） |
| 每 bucket 一鎖 | 對 | 好（不同 bucket 可平行） |

benchmark 要求（2 threads 至少 1.25x）基本上需要 per-bucket lock。

### 5. Barrier 的 `round` 計數

condition variable 沒有記憶，若 `broadcast` 早於 `wait`，訊號就丟了。沒有 `round` 時，thread 可能用到舊狀態跨輪等待。

`round` 是 世代 counter：thread 先拍下 `my_round`，等到 `b.round != my_round` 才放行。

---

## 常見陷阱

### 1. `sp` 設在 stack 底部而不是頂部

RISC-V stack 向下長（高位址往低位址）。`sp` 初值要在配置區最高位址。

錯誤：
```c
t->context.sp = (uint64)t->stack;           // 底部，第一次 push 就越界
```

正確：
```c
t->context.sp = (uint64)(t->stack + STACK_SIZE);  // 頂部
```

### 2. `ra` 沒設或設成 0

若 `thread_switch` 執行 `ret` 時 `ra=0`，新 thread 會跳到位址 0，立即崩潰。

`thread_create` 必須在首次排程前設定 `context.ra = (uint64)func`。

### 3. thread_switch.S 少存/少還原暫存器

只要漏掉任一 callee-saved register，thread 切走再回來時狀態就會壞。

offset 必須精準對齊 `struct context`：
```text
ra  →  0
sp  →  8
s0  → 16
s1  → 24
...
s11 → 104
```

### 4. Hash table 還有非 bucket-local 共享狀態

若 `put` 還會改到全域計數器或全域串列，只有 per-bucket lock 不夠，該共享狀態仍會 race。

鎖要小，但要涵蓋所有共享可變狀態。

### 5. Condition wait 用 `if` 而非 `while`

`pthread_cond_wait` 可能 spurious wakeup（即使沒有真正 broadcast 也返回）。

錯誤：
```c
if (b.round == my_round)
    pthread_cond_wait(&b.cond, &b.lock);
```

正確：
```c
while (b.round == my_round)
    pthread_cond_wait(&b.cond, &b.lock);
```

### 6. Barrier 的 `count` 在 broadcast 後才重設

若 `count` 重設太晚，快 thread 可能先回來把下一輪狀態污染。

要在持鎖下、`broadcast` 前先做：

```c
b.round++;
b.count = 0;               // 先重設
pthread_cond_broadcast(&b.cond);
```

### 7. thread_switch 前沒先更新 `current_thread`

`thread_switch` 會切 stack。若回到新 stack 後 `current_thread` 還指舊 thread，後續邏輯會操作錯對象。

```c
// 正確順序：
struct thread *old = current_thread;
current_thread = next;          // switch 前先更新
thread_switch(&old->context, &next->context);
// 現在在 next 的 stack 上，current_thread 也是正確的
```

---

## Uthread 與 Kernel Scheduler 的對照

uthread 本質上對應 xv6 kernel 的 `swtch()`：

| | uthread | xv6 kernel |
|--|---------|------------|
| Context struct | user space 的 `struct context` | `kernel/proc.h` 的 `struct context` |
| Switch 函式 | user assembly 的 `thread_switch()` | `kernel/swtch.S` 的 `swtch()` |
| 排程 | `thread_schedule()`（round-robin） | `scheduler()`（round-robin） |
| 保存暫存器 | `ra`, `sp`, `s0`–`s11` | 相同 |
| Stack | 內嵌 char array | `p->kstack`（kernel page） |
| Preemption | 無（cooperative） | 有（timer interrupt → `yield()`） |

最大差異是：kernel thread 會被 timer interrupt 搶占；uthread 只有 thread 自己呼叫 `thread_yield()` 才會切換。
