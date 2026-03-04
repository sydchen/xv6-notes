# xv6 Locks

https://pdos.csail.mit.edu/6.1810/2025/labs/lock.html

## 目錄
1. [背景：為什麼 Lock Contention 很傷](#背景為什麼-lock-contention-很傷)
2. [Lab 任務總覽](#lab-任務總覽)
3. [任務 1：Per-CPU 記憶體配置器](#任務-1per-cpu-記憶體配置器)
4. [任務 2：Read-Write Spinlock](#任務-2read-write-spinlock)
5. [Code Sketches](#code-sketches)
6. [關鍵設計決策](#關鍵設計決策)
7. [常見陷阱](#常見陷阱)

---

## 背景：為什麼 Lock Contention 很傷

xv6 原本的記憶體配置器只有一個全域 free list：

```text
kmem.lock ──► freelist → page → page → page → ...
```

在多核心環境下，所有 CPU 的 `kalloc()` / `kfree()` 都要搶同一把鎖。CPU 會花很多時間 spin 等待，而不是做真正工作，這就是 **lock contention**。

典型改善方式是：把共享資源做 **sharding**，讓不同 CPU 盡量獨立操作。

---

## Lab 任務總覽

| 任務 | 問題 | 解法 |
|------|------|------|
| 記憶體配置器 | 單一鎖造成全 CPU 競爭 | Per-CPU free list + stealing |
| Read-write lock | 一般 spinlock 不分讀寫 | 多讀者/單寫者 + writer priority |

兩個任務彼此獨立，順序可自由。

---

## 任務 1：Per-CPU 記憶體配置器

### 核心概念

每個 CPU 有自己的 free list，也有自己的鎖：

```text
CPU 0: kmem[0].lock ──► freelist → page → page → ...
CPU 1: kmem[1].lock ──► freelist → page → page → ...
CPU 2: kmem[2].lock ──► freelist → page → page → ...
CPU 3: kmem[3].lock ──► freelist → page → page → ...
```

正常路徑下，CPU 0 只碰 `kmem[0]`，不會碰到其他 CPU 的鎖。

### Stealing

若某 CPU 的本地 list 空了，就去別的 CPU 偷一頁：

```text
Fast path: own list 有頁面 → 直接回傳
Slow path: own list 空 → 掃其他 CPU → 從第一個非空 list 拿一頁
```

穩定狀態下 slow path 很少走到，所以偶發的跨 CPU 上鎖成本可接受。

### push_off / pop_off

`cpuid()` 會回傳當前 CPU 編號。但在 `cpuid()` 與使用結果之間，thread 可能被 scheduler 搬到另一顆 CPU。避免方式是暫時關中斷：

```c
push_off();          // 關中斷（避免 migration）
int c = cpuid();     // 此時 CPU 編號可安全使用
// ... use kmem[c] ...
pop_off();           // 開中斷
```

`push_off` / `pop_off` 可巢狀，內部是 depth counter，不是單純 on/off。

### 初始分布

`kinit()` 只呼叫一次 `freerange()`。因此初始頁面會先落在某一顆 CPU 的 list（取決於誰在跑 `kinit()`）。其他 CPU 一開始可能是空的，第一次配置時會偷頁。這是正常行為，之後會自然再分布。

---

## 任務 2：Read-Write Spinlock

### 動機

普通 spinlock 不分讀寫，任何時刻只能一個持有者。對純讀場景來說太保守。

rwlock 可以放寬：

| 狀態 | 可進入者 |
|------|----------|
| Idle | 任一讀者或一個寫者 |
| Readers holding | 可再進讀者（並行），不可進寫者 |
| Writer holding | 其他都不可進 |

當讀多寫少時（例如 cache、routing table、配置資料）特別有價值。

### Writer Priority

若沒有特別處理，讀者持續湧入會讓寫者飢餓（reader count 很難歸零）。

**Writer priority** 的規則是：只要有寫者在等，新的讀者就先擋住。已在內部的讀者可跑完，但不再放新讀者進來。

```text
readers_waiting  = 0    (讀者視角：我能進嗎)
writers_waiting  = 0    (寫者公告：我在等)
writer_active    = 0/1  (是否已有寫者持有)

讀者可進條件：writer_active == 0 且 writers_waiting == 0
寫者可進條件：writer_active == 0 且 readers == 0
```

### 狀態轉移

```text
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

### 內部鎖（Inner Lock）

rwlock 內部還有一把 spinlock，僅用來保護計數欄位更新；不會在真正的讀寫臨界區持有。

```c
struct rwspinlock {
    struct spinlock lock;     // 保護下面欄位
    uint readers;             // 活躍讀者數
    uint writers_waiting;     // 等待中的寫者數
    uint writer_active;       // 1 表示寫者持有中
};
```

spin-wait 會在 loop 中反覆釋放/重拿這把內部鎖（busy-wait），對這種短臨界區是合理做法。

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
        memset(page, 1, PGSIZE);   // 填垃圾值，幫助抓 use-after-free
    return (void *)page;
}
```

### 2. Per-CPU kfree()

```c
void kfree(void *pa) {
    struct run *page = (struct run *)pa;
    memset(page, 1, PGSIZE);   // 抓 dangling reference

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

    // 有寫者 active 或 waiting 都要擋（writer priority）
    while (rwlk->writer_active || rwlk->writers_waiting) {
        release(&rwlk->lock);
        acquire(&rwlk->lock);   // spin
    }

    rwlk->readers++;
    release(&rwlk->lock);
    // 此時讀鎖視為持有中，但內部鎖已釋放
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

    // 宣告自己在等待，阻擋新讀者
    rwlk->writers_waiting++;

    // 等到沒有讀者且沒有其他寫者
    while (rwlk->readers > 0 || rwlk->writer_active) {
        release(&rwlk->lock);
        acquire(&rwlk->lock);   // spin
    }

    rwlk->writers_waiting--;
    rwlk->writer_active = 1;
    release(&rwlk->lock);
    // 寫鎖持有中
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

## 關鍵設計決策

### 1. 一次偷一頁

本地 list 空時，從其他 CPU 一次只偷 **一頁**，而不是整串。

| 策略 | 區域性 | 每次偷取成本 |
|------|--------|--------------|
| 偷一頁 | 若很快釋放，頁面回到本地機率高 | O(1) |
| 偷半串 | 攤銷較好 | O(n)，且持有受害者鎖更久 |

在 xv6 這個規模，一次一頁比較簡潔且足夠。

### 2. 鎖名稱要以 `kmem` 開頭

`kalloctest` 會檢查 lock 名稱前綴。每個 per-CPU lock 名稱都要是 `"kmem"`（例如 `kmem0`、`kmemX`）。名稱不符即使邏輯正確也會測試失敗。

### 3. writers_waiting 只擋新讀者

writer priority 的重點是擋 **新進** 讀者，不是把已進入的讀者踢出去。已在臨界區的讀者跑完就離開。

所以寫者仍可能等一下，但因為不再放新讀者進來，理論上最終可進入。

### 4. Spin-wait 期間要釋放內部鎖

rwlock 的等待 loop 必須先釋放內部鎖再重拿：

```c
while (condition) {
    release(&rwlk->lock);
    acquire(&rwlk->lock);   // 重拿後再檢查
}
```

如果等待時一直握著內部鎖，其他執行緒無法更新狀態，系統就沒進展。

### 5. writer_active 用 flag 就夠

同一時間只會有一個寫者持有，所以 `writer_active` 用 0/1 flag 即可。release 時加防呆檢查仍很有用：

```c
void write_release(struct rwspinlock *rwlk) {
    if (rwlk->writer_active == 0)
        panic("write_release: not locked");
    // ...
}
```

---

## 常見陷阱

### 1. 沒關中斷就呼叫 cpuid()

**錯誤：**
```c
int c = cpuid();           // 這裡可能被中斷搬家
acquire(&kmem[c].lock);    // 接下來可能操作錯誤 CPU 的 list
```

**正確：**
```c
push_off();
int c = cpuid();           // 中斷關閉期間，CPU 不會被切走
acquire(&kmem[c].lock);
// ...
pop_off();
```

`cpuid()` 與 `acquire()` 間若發生 migration，鎖到的 CPU 與實際執行 CPU 會不一致。

### 2. spin loop 中一直持有內部鎖

**錯誤：**
```c
acquire(&rwlk->lock);
while (rwlk->writer_active) {
    // 持鎖空轉，其他人無法更新 writer_active
}
```

**正確：**
```c
acquire(&rwlk->lock);
while (rwlk->writer_active) {
    release(&rwlk->lock);   // 先放給別人更新
    acquire(&rwlk->lock);   // 再重拿
}
```

### 3. read_acquire() 忘記檢查 writers_waiting

**錯誤：**
```c
while (rwlk->writer_active) {
    // ...
}
```

寫者可能已在等（`writers_waiting > 0`）但尚未 active。若不檢查 waiting，新的讀者會一直進，寫者就飢餓。

**正確：**
```c
while (rwlk->writer_active || rwlk->writers_waiting) {
    // ...
}
```

### 4. write_acquire() 忘記 writers_waiting--

```c
void write_acquire(struct rwspinlock *rwlk) {
    acquire(&rwlk->lock);
    rwlk->writers_waiting++;    // 宣告等待

    while (rwlk->readers > 0 || rwlk->writer_active) {
        release(&rwlk->lock);
        acquire(&rwlk->lock);   // spin
    }

    rwlk->writers_waiting--;    // 這行很常漏
    rwlk->writer_active = 1;
    release(&rwlk->lock);
}
```

若漏掉遞減，`writers_waiting` 會只增不減，導致讀者長期被擋。

### 5. 沒有掃完整個其他 CPU 集合

```c
// 錯誤：只試 CPU 0
acquire(&kmem[0].lock);
// steal from kmem[0]
release(&kmem[0].lock);
```

```c
// 正確：掃所有其他 CPU
for (int i = 0; i < NCPU; i++) {
    if (i == c) continue;
    // try kmem[i] ...
}
```

若 CPU 0 也空，錯誤版本會太早回傳 NULL。

### 6. 同時持有兩把 kmem 鎖

```c
// 危險：持有 kmem[c].lock 時又拿 kmem[i].lock
acquire(&kmem[c].lock);
struct run *r = kmem[c].freelist;
// list empty, try to steal
acquire(&kmem[i].lock);   // 兩顆 CPU 互相這樣做時容易死鎖
```

**正確：** 先放本地鎖，再拿受害者鎖。上面的實作就是 fast path 先 release，slow path 才逐一拿 `kmem[i].lock`。

---

## 改善怎麼量測

這個 lab 的重點是可觀測的 contention 下降。`kalloctest` 會報 lock acquire / wait 計數：

```text
Before (single list):
  kmem: #acquire=100000, #wait=72000   ← wait 很高

After (per-CPU lists):
  kmem0: #acquire=30000, #wait=5
  kmem1: #acquire=28000, #wait=3
  ...                                  ← contention 幾乎消失
```

wait count 下降，代表 CPU 大多只碰自己的 free list；這就是 sharding 的實際收益。
