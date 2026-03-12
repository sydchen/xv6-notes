# xv6 Page Tables

https://pdos.csail.mit.edu/6.1810/2025/labs/pgtbl.html

## 目錄
1. [背景：RISC-V 頁表](#背景risc-v-頁表)
2. [Lab 任務總覽](#lab-任務總覽)
3. [任務 1：觀察使用者行程頁表](#任務-1觀察使用者行程頁表)
4. [任務 2：加速系統呼叫（USYSCALL）](#任務-2加速系統呼叫usyscall)
5. [任務 3：列印頁表（vmprint）](#任務-3列印頁表vmprint)
6. [任務 4：Superpages](#任務-4superpages)
7. [Code Sketches](#code-sketches)
8. [設計決策](#設計決策)
9. [常見陷阱](#常見陷阱)

---

## 背景：RISC-V 頁表

### 三層頁表結構

RISC-V sv39 使用三層頁表。每層都是一頁 4 KB，內含 512 個 PTE（每個 8 bytes）：

```text
Virtual Address (39 bits):
  [L2 index: 9 bits][L1 index: 9 bits][L0 index: 9 bits][page offset: 12 bits]

Translation walk:
  CR3 / satp → L2 page table
    [L2 index] → L1 page table
      [L1 index] → L0 page table
        [L0 index] → physical page
          + offset → final physical address
```

每個 PTE 包含 physical page number (PPN) 與旗標位元：

| Flag | Bit | 意義 |
|------|-----|------|
| V | 0 | Valid，條目有效 |
| R | 1 | 可讀 |
| W | 2 | 可寫 |
| X | 3 | 可執行 |
| U | 4 | 使用者可存取 |
| G | 5 | 全域映射 |
| A | 6 | 已存取 |
| D | 7 | 已修改 |

若 R/W/X 全為 0，PTE 指向下一層頁表（non-leaf）；若 R/W/X 任一為 1，是 leaf，直接映射到實體頁。

### 使用者虛擬位址的固定高位區

xv6 在每個 user address space 高位區都映射幾個固定頁面（在 user program 上方）：

```text
MAXVA
  TRAMPOLINE   (kernel trampoline code, exec/ecall path)
  TRAPFRAME    (per-process，保存 trap 時的 user registers)
  USYSCALL     (本 lab 新增，共享唯讀頁)
  ...
  user stack / heap / text / data
0x0
```

### Superpages（2 MB 頁）

一般 leaf PTE 在 L0，覆蓋 4 KB。但 leaf 也可以放在 L1，覆蓋 512 × 4 KB = 2 MB，這就是 superpage。

硬體 walk 流程不變；差別只在 L1 PTE 直接設成 leaf（R/W/X 置位），而不是往下指到 L0。

---

## Lab 任務總覽

| Task | 你要實作什麼 | 難度 |
|------|---------------|------|
| Inspect page table | 解析 `pgtbltest` 輸出、說明 PTE 意義 | Easy |
| USYSCALL | 共享唯讀頁，加速 `getpid()` | Easy |
| vmprint() | 遞迴列印頁表 | Easy |
| Superpages | 2 MB 分配、複製、降級（demotion） | Moderate/Hard |

---

## 任務 1：觀察使用者行程頁表

### 目標

執行 `pgtbltest`，說明每個頁表項（PTE）代表什麼：虛擬位址區、權限旗標、對應實體頁。

### 要看什麼

剛 `exec` 後的使用者行程有可預期映射：

```text
page table 0x0000000087f6e000
 ..0: pte 0x21fda801 pa 0x87f6a000    ← L2 PTE，指到 L1 table
 .. ..0: pte 0x21fd9c01 pa 0x87f67000  ← L1 PTE，指到 L0 table
 .. .. ..0: pte 0x21fdac1f pa 0x87f6b000  ← text page (R, X, U, V)
 .. .. ..1: pte 0x21fda41f pa 0x87f69000  ← data page
 ...
 ..255: pte 0x...         pa 0x...     ← TRAMPOLINE / TRAPFRAME
```

non-leaf PTE 的 R=W=X=0（只有 V=1）。leaf PTE 至少有一個 R/W/X 被設。`U` 位元決定 user mode 是否可存取。

### 關鍵觀察

虛擬位址連續不代表實體位址連續。頁表的角色就是把離散的實體 frame 映射成連續的虛擬空間。

---

## 任務 2：加速系統呼叫（USYSCALL）

### 問題

每次 syscall（即使像 `getpid()` 這種簡單查詢）都要 user → kernel → user 來回，成本在 trap 開銷（保存/還原暫存器、切頁表等）。

### 解法：共享唯讀頁

在固定虛擬位址（`USYSCALL`）映射一頁，kernel 可寫、user 可讀，讓讀取不必 trap：

```text
Kernel side (proc.c):
  p->usyscall = kalloc();       // 分配實體頁
  p->usyscall->pid = p->pid;   // 寫入 PID
  mappages(pagetable, USYSCALL, PGSIZE,
           (uint64)p->usyscall,
           PTE_R | PTE_U);      // user 只能讀

User side (ulib.c):
  int getpid(void) {
      struct usyscall *u = (struct usyscall *)USYSCALL;
      return u->pid;  // 直接讀記憶體，不用 trap
  }
```

讀路徑不需要 `ecall`。kernel 會在行程中繼資料變動時維護 `p->usyscall->pid`。

### 生命週期

| 事件 | 動作 |
|------|------|
| `allocproc()` | `kalloc()` 一頁並填 `pid` |
| `proc_pagetable()` | `mappages(USYSCALL, PTE_R|PTE_U)` |
| `proc_freepagetable()` | `uvmunmap(USYSCALL, ...)` |
| `freeproc()` | `kfree(p->usyscall)` |

USYSCALL 映射必須先解除，再釋放實體頁，否則該 VA 可能懸掛到被重用的記憶體。

### 哪些 syscall 也適合？

適合回傳唯讀、低變動 kernel 狀態的 syscall：`getppid()`、`uptime()`、CPU 數量等。只要 kernel 能主動把值更新到共享頁，此模式就成立。

---

## 任務 3：列印頁表（vmprint）

### 目標

實作 `vmprint(pagetable_t pagetable)`，遞迴列出所有 valid PTE，縮排表達層級：

```text
page table 0x0000000087f6e000
 ..0: pte 0x... pa 0x...        ← level 2
 .. ..0: pte 0x... pa 0x...     ← level 1
 .. .. ..0: pte 0x... pa 0x...  ← level 0 / leaf
```

### 演算法

```c
void vmprintwalk(pagetable_t pt, int level) {
    for (int i = 0; i < 512; i++) {
        pte_t pte = pt[i];
        if (!(pte & PTE_V))
            continue;

        // 縮排：(level+1) 組 " .."
        for (int j = 0; j <= level; j++)
            printf(" ..");
        uint64 pa = PTE2PA(pte);
        printf("%p: pte %lx pa %lx\n", &pt[i], pte, pa);

        // Non-leaf: R=W=X=0，遞迴到下一層
        if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)
            vmprintwalk((pagetable_t)pa, level + 1);
    }
}

void vmprint(pagetable_t pagetable) {
    printf("page table %p\n", pagetable);
    vmprintwalk(pagetable, 0);
}
```

### 何時呼叫？

通常在第一次 `exec` 後呼叫一次，觀察新行程頁表。也常用於 `panic()` 除錯。

---

## 任務 4：Superpages

### 動機

正常映射 2 MB 需要：1 個 L2 table + 1 個 L1 table + 1 個 L0 table + 512 個 L0 PTE。superpage 可把這段換成 1 個 L1 leaf PTE，降低頁表與 TLB 成本。

### 實作計畫

1. 開機時 **預留** 一批 2 MB 對齊實體區塊（`kinit`）
2. 在 `uvmalloc` 遇到 2 MB 對齊且剩餘 >= 2 MB 時，優先 `superalloc`
3. 用 L1 leaf PTE 映射（`set_superpage`）
4. `uvmcopy` 支援整個 superpage 複製（`fork`）
5. `uvmunmap` 支援整個 superpage 釋放
6. 若只釋放 superpage 的一部分，先做 demotion 成 512 個 4 KB 頁

### Superpage 記憶體池

開機時，在加入一般 4 KB free list 前，先切出固定數量的 2 MB 對齊區塊：

```c
void kinit() {
    // 找 kernel end 之後第一個 2MB 對齊位址
    uint64 aligned_start = SUPERPGROUNDUP((uint64)end);

    // 最多預留 16 個 superpages（總共 32 MB）
    for (each 2MB region starting at aligned_start) {
        add to super_kmem.superlist;
    }

    // 剩餘頁面交給一般 kmem.freelist
    freerange(end, aligned_start);
    freerange(aligned_start + reserved_size, PHYSTOP);
}
```

### 安裝 Superpage 映射

superpage leaf 在 L1，不在 L0。安裝流程：

```c
int set_superpage(pagetable_t pagetable, uint64 va, uint64 pa, int perm) {
    // va, pa 都必須 2MB 對齊
    pte_t *pte2 = &pagetable[PX(2, va)];       // L2

    // 缺 L1 table 就配置
    if (!(*pte2 & PTE_V)) {
        pagetable_t level1 = kalloc();
        *pte2 = PA2PTE(level1) | PTE_V;
    }

    pagetable_t level1 = PTE2PA(*pte2);
    pte_t *pte1 = &level1[PX(1, va)];          // L1

    // 設成 leaf（R/W/X 有設就會被硬體視為 leaf）
    *pte1 = PA2PTE(pa) | perm | PTE_V;

    // 不配置 L0 table
    return 0;
}
```

### 偵測是否為 Superpage

判斷某 VA 是否由 superpage 映射：

```c
int is_superpage_pte(pagetable_t pagetable, uint64 va) {
    pte_t *pte2 = &pagetable[PX(2, va)];
    if (!(*pte2 & PTE_V) || PTE_LEAF(*pte2))
        return 0;   // L2 不存在或 L2 本身是 leaf

    pagetable_t level1 = PTE2PA(*pte2);
    pte_t *pte1 = &level1[PX(1, va)];

    // L1 valid leaf == superpage
    return (*pte1 & PTE_V) && PTE_LEAF(*pte1);
}
```

### Demotion：切分 Superpage

只釋放 superpage 一部分時，不能直接清掉 L1 leaf，否則剩下資料也會不見。要先 demote：

```c
int demote_superpage(pagetable_t pagetable, uint64 va, pte_t *superpte) {
    uint64 superpa = PTE2PA(*superpte);
    uint flags = PTE_FLAGS(*superpte);

    // 新配一張 L0 頁表
    pagetable_t level0 = kalloc();

    // 把 superpage 拆成 512 個 4KB 頁
    for (int i = 0; i < 512; i++) {
        char *mem = kalloc();
        memmove(mem, (char*)(superpa + i * PGSIZE), PGSIZE);
        level0[i] = PA2PTE(mem) | flags;
    }

    // 用 L0 頁表指標取代 L1 leaf
    *superpte = PA2PTE(level0) | PTE_V;   // 僅 V -> non-leaf

    // 釋放原始 2MB 區塊
    superfree((void*)superpa);
    return 0;
}
```

demote 完後，該區段看起來就和一般 4 KB 映射完全一致。

---

## Code Sketches

### 1. usyscall 結構與映射

```c
// kernel/memlayout.h
struct usyscall {
    int pid;   // 可由 user space 讀取的 process ID
};

// kernel/proc.c — allocproc()
p->usyscall = (struct usyscall *)kalloc();
p->usyscall->pid = p->pid;

// kernel/proc.c — proc_pagetable()
mappages(pagetable, USYSCALL, PGSIZE,
         (uint64)p->usyscall,
         PTE_R | PTE_U);

// kernel/proc.c — proc_freepagetable()
uvmunmap(pagetable, USYSCALL, 1, 0);   // 只解除映射，不釋放實體頁

// kernel/proc.c — freeproc()
kfree((void*)p->usyscall);
p->usyscall = 0;
```

### 2. vmprint() — 分層頁表輸出

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
    if (r) memset(r, 5, SUPERPGSIZE);   // 填垃圾值
    return (void *)r;
}

void superfree(void *pa) {
    // 驗證對齊與範圍
    memset(pa, 1, SUPERPGSIZE);
    struct run *r = (struct run *)pa;
    acquire(&super_kmem.lock);
    r->next = super_kmem.superlist;
    super_kmem.superlist = r;
    release(&super_kmem.lock);
}
```

### 4. uvmalloc — 先嘗試 superpage

```c
for (a = oldsz; a < newsz; a += sz) {
    // 2MB 對齊且剩餘 >=2MB 時，先嘗試 superpage
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
        // superalloc 失敗 -> 回退到 4KB 配置
    }
    sz = PGSIZE;
    // ... normal kalloc + mappages ...
}
```

### 5. uvmunmap — 處理 superpage

```c
for (a = va; a < va + npages * PGSIZE; a += sz) {
    sz = PGSIZE;
    uint64 base_va = SUPERPGROUNDDOWN(a);

    if (is_superpage_pte(pagetable, base_va)) {
        pte_t *pte1 = /* get L1 PTE for base_va */;
        uint64 end_va = va + npages * PGSIZE;

        if (a == base_va && end_va >= base_va + SUPERPGSIZE) {
            // 整個 superpage 一次解除
            sz = SUPERPGSIZE;
            if (do_free) superfree((void *)PTE2PA(*pte1));
            *pte1 = 0;
            continue;
        } else {
            // 部分解除：先 demote
            if (do_free)
                demote_superpage(pagetable, base_va, pte1);
            else { *pte1 = 0; sz = SUPERPGSIZE; continue; }
        }
    }

    // 一般 4KB 邏輯...
}
```

---

## 設計決策

### 1. USYSCALL：user 唯讀，kernel 可寫

只給 `PTE_R | PTE_U`（不給 `PTE_W`）能確保 user 只能讀共享頁，不能寫。kernel 在 supervisor mode 可更新 `pid`。

這是刻意的安全邊界：快路徑能成立，是因為內容由 kernel 控制。

### 2. vmprint：用 R/W/X 區分 leaf 與 non-leaf

在 RISC-V，判斷 leaf 的正確方法是看 R/W/X 是否有任一置位。`V=1` 且 R=W=X=0 一定是下一層頁表指標。

不要只靠層級判斷是否遞迴，因為 superpage leaf 會出現在 L1。

### 3. Superpage pool：開機預留，不動態搜尋

在一般 free list 上動態找 2 MB 對齊區塊會遇到碎片與對齊成本。比較簡單的作法是開機時先預留固定 2 MB 區塊，掛到獨立 `superlist`。

取捨是容量固定（16 個 superpages = 32 MB）。用完後 `superalloc()` 回 0，`uvmalloc` 自動回退到一般 4 KB 映射。

### 4. Demotion：先複製，再釋放

只釋放 2 MB 區段的一部分時，剩餘資料必須保留。demotion 流程：
1. 配 512 個 4 KB 頁
2. 從原 superpage 逐塊複製
3. 建好 L0 頁表
4. 最後才釋放原 2 MB 區塊

順序不能反，先 free 會讀到已釋放記憶體。

### 5. Superpage 對齊：VA/PA 都要 2 MB 對齊

L1 leaf 映射時，VA 與 PA 都必須 2 MB 對齊，否則 superpage 內部 offset 會錯。

`SUPERPGROUNDUP` / `SUPERPGROUNDDOWN` 就是用來強制這件事。

---

## 常見陷阱

### 1. 忘記在 proc_freepagetable 解除 USYSCALL

若先在 `freeproc` 釋放實體頁，後面頁表走訪還看到 USYSCALL 映射，就可能解參考到垃圾。

正確順序：
```c
proc_freepagetable(p->pagetable, p->sz);  // 這裡先 unmap USYSCALL
p->pagetable = 0;
// ...
kfree(p->usyscall);                        // 再 free 實體頁
```

`uvmunmap(pagetable, USYSCALL, 1, 0)` 應放在 `proc_freepagetable`。

### 2. vmprint 錯把 leaf 當頁表遞迴

L1 superpage 也是 leaf（`R/W/X != 0`）。若只依層級遞迴，會把資料頁當成頁表解析，輸出錯誤甚至 panic。

錯誤：
```c
if (level < 2)   // 在 level 0/1 一律遞迴
    vmprintwalk((pagetable_t)pa, level + 1);
```

正確：
```c
if ((pte & (PTE_R | PTE_W | PTE_X)) == 0)   // non-leaf 才遞迴
    vmprintwalk((pagetable_t)pa, level + 1);
```

### 3. Demotion 時先 free superpage

superpage 是複製來源。若在 512 塊複製完成前先 free，資料一定壞。

### 4. do_free=0 的 partial unmap 處理錯誤

`uvmunmap(..., do_free=0)`（例如 exec 清舊空間）不該做 demotion。這時應清掉對應 PTE。若 demote 但不 free，會配置 512 個新頁並漏掉原 superpage。

### 5. is_superpage_pte 用了未對齊位址

走訪區間時，同一個 2 MB 區段內任意位址都指向同一 superpage。呼叫 `is_superpage_pte` 前要先 round down：

```c
uint64 base_va = SUPERPGROUNDDOWN(a);
if (is_superpage_pte(pagetable, base_va)) { ... }
```

直接用未對齊 `a` 可能檢查到錯誤 L2/L1 index。

### 6. USYSCALL 權限設成可寫

USYSCALL 應該 user 可讀、不可寫。若加 `PTE_W`，user 可改 kernel 管理資料。

```c
// Correct:
mappages(pagetable, USYSCALL, PGSIZE, (uint64)p->usyscall, PTE_R | PTE_U);

// Wrong (user 可寫):
mappages(pagetable, USYSCALL, PGSIZE, (uint64)p->usyscall, PTE_R | PTE_W | PTE_U);
```

---

## 各部分怎麼串起來

```text
sbrk(2MB)
  → sys_sbrk → growproc(2MB)
      → uvmalloc(pagetable, oldsz, newsz, xperm)
          if (2MB-aligned && 2MB remaining):
              mem = superalloc()   ← 來自開機預留池
              set_superpage(pagetable, va, pa, flags)
                  ← L2 non-leaf → L1 leaf PTE (沒有 L0!)
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
