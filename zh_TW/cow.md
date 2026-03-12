# xv6 Copy-on-Write Fork

https://pdos.csail.mit.edu/6.1810/2025/labs/cow.html

## 目錄
1. [什麼是 Copy-on-Write Fork？](#什麼是-copy-on-write-fork)
2. [核心觀念](#核心觀念)
3. [實作架構](#實作架構)
4. [偽程式碼](#偽程式碼)
5. [設計決策](#設計決策)
6. [常見陷阱](#常見陷阱)

---

## 什麼是 Copy-on-Write Fork？

傳統的 `fork()` 會把父行程的整份記憶體完整複製給子行程。這其實很浪費：`fork()` 幾乎總是接著 `exec()`，而 `exec()` 會把剛剛複製的頁面全部丟掉；就算沒有 `exec()`，子行程也不一定真的會去寫那些頁面。

COW 的做法是延後複製，等到真的需要寫入時才處理。`fork()` 當下，父子行程先共享同一批實體頁面，兩邊都標成唯讀。哪一邊嘗試寫入，就觸發 page fault，fault handler 再幫它建立私有副本。到這一步才真正複製記憶體。

可以想成雲端共用文件：大家先看同一份，只有有人開始編輯時，系統才幫他分出自己的版本。

---

## 核心觀念

### 1. PTE_COW 旗標

要把 COW 頁面和「本來就應該唯讀」的頁面區分開，需要一個標記表示：「這頁其實可寫，但目前在共享，所以寫之前要先複製。」

RISC-V 的 PTE 有 RSW（Reserved for Software）bits，硬體不會使用，軟體可以自由定義。我們設定：

```
PTE_COW = bit 8（RSW 其中一個 bit）
```

COW 頁面會有 `PTE_W = 0`（從硬體角度看是禁止寫入）加上 `PTE_COW = 1`（軟體標記：這是 COW，不是天生唯讀）。發生寫入 fault 時，看 `PTE_COW` 就知道是要做複製，還是直接 kill process（真的違反唯讀保護）。

### 2. 參考計數（Reference Counting）

多個行程共享同一個實體頁面時，要知道什麼時候可以安全釋放。只有在沒有任何行程再指到該頁面時，才可以 free。

我們對每個實體頁面維護一個參考計數：

```
page_ref[physical_addr / PGSIZE] = 指向這個頁面的 PTE 數量
```

- `kalloc()` → ref count 設為 1
- `fork()` 共享頁面 → ref count 加 1
- `kfree()` → ref count 減 1；只有變成 0 才真的釋放

### 3. COW Page Fault

當行程寫入 COW 頁面：

1. 因為 `PTE_W` 被清掉，硬體觸發 write page fault
2. fault handler 發現 `PTE_COW` 有設 → 這是 COW fault
3. 分兩種情況：
   - ref count == 1：只剩這個行程在用，直接恢復可寫，不用複製
   - ref count > 1：還有人共享，配置新頁面、複製內容、更新 PTE

### 4. 記憶體配置（與一般 fork 相同）

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
|     DATA          |   ← 一開始這裡的 COW 頁面會共享
+-------------------+
|     TEXT          |   ← 原本就唯讀；不需要 COW
+-------------------+  ← 0x0
```

Text（程式碼）頁面本來就唯讀，天然可共享，不需要額外的 COW 處理。

---

## 實作架構

### 元件

```
User Space
  ↓ fork() / write to shared page
Kernel System Calls
  ↓ uvmcopy() — 在 fork 時共享頁面
Page Table Management (vm.c)
  ↓ page fault
Page Fault Handler (vmfault)
  ↓ reference counting
Physical Memory Allocator (kalloc.c)
```

### 資料結構

```
Physical Memory Reference Counting:
  pageref.count[PHYSTOP / PGSIZE]  — 每個 page frame 一個 counter
  pageref.lock                      — 併發存取用 spinlock
```

### 主要流程

**fork() 流程：**
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

**對 COW 頁面寫入時：**
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

**kfree() 流程：**
```
kfree(page):
  decrement ref count
  if ref count > 0:
      return (other processes still use this page)
  else:
      actually free the page
```

---

## 偽程式碼

### 1. allocator 內的 reference counting

```c
struct pageref {
    struct spinlock lock;
    int count[PHYSTOP / PGSIZE];
};

void *kalloc(void) {
    void *page = pop from freelist;
    if (page)
        pageref.count[(uint64)page / PGSIZE] = 1;  // 新頁面起始 ref=1
    return page;
}

void kfree(void *page) {
    acquire(&pageref.lock);
    int idx = (uint64)page / PGSIZE;
    if (pageref.count[idx] > 1) {
        pageref.count[idx] -= 1;
        release(&pageref.lock);
        return;                             // 還有人引用，不 free
    }
    pageref.count[idx] = 0;
    release(&pageref.lock);
    // 真正把頁面放回 freelist
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
            continue;                       // lazy allocation 頁面，略過

        uint64 pa = PTE2PA(*pte);
        uint64 flags = PTE_FLAGS(*pte);

        if (flags & PTE_W) {
            flags = (flags | PTE_COW) & ~PTE_W;  // 標成 COW、清除寫入權限
            *pte = PA2PTE(pa) | flags;             // 父行程 PTE 也要更新
        }

        if (mappages(child_pagetable, va, PGSIZE, pa, flags) < 0)  // 子行程映射同一個實體頁面
            goto on_error;
        kref_inc((void *)pa);              // ref 數 +1
    }
    return 0;

on_error:
    uvmunmap(child_pagetable, 0, va / PGSIZE, 1);  // 釋放子行程已映射的所有頁面
    return -1;
}
```

注意父行程的 PTE 也要更新。如果只把子行程設成唯讀、父行程還是可寫，那父行程依然能直接改共享頁面，兩邊就失去保護了。

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
            // 只剩自己在用，直接恢復寫入權限
            *pte |= PTE_W;
            *pte &= ~PTE_COW;
            return pa;
        }

        // 還有其他引用，必須複製
        void *new_page = kalloc();
        if (new_page == 0)
            return 0;                       // OOM → kill process

        memmove(new_page, (void *)pa, PGSIZE);  // 複製舊內容

        flags = (flags | PTE_W) & ~PTE_COW;
        *pte = PA2PTE((uint64)new_page) | flags;  // PTE 改指向新的私有頁面

        kfree((void *)pa);                  // 釋放舊頁面的這一份引用
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

### 4. copyout() — Kernel 寫入 User Space

```c
int copyout(pagetable_t pagetable, uint64 dst_va, char *src, uint64 len) {
    while (len > 0) {
        uint64 va = PGROUNDDOWN(dst_va);
        pte_t *pte = walk(pagetable, va, 0);

        // Kernel 寫到 COW 頁面時，也要先觸發複製
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

`copyout()` 需要特別處理，因為 kernel 把資料拷貝到 user space（例如 `read()` 系統呼叫）時是直接寫記憶體，不會走硬體 page fault 流程。這條路徑必須自己檢查並處理 COW。

### 5. Process Exit

```c
void exit(int status) {
    // ... close file descriptors, etc. ...

    // 釋放整個行程記憶體；kfree 會自動處理 reference counting
    uvmfree(p->pagetable, p->sz);
    // 每釋放一頁都會呼叫 kfree()，讓 ref count 減一
    // 只有 ref count 歸零才會真正釋放實體頁面
}
```

`exit()` 不需要額外做 COW 特判，reference-counted 的 `kfree()` 會把事情處理好。

---

## 設計決策

### 1. 用 RSW Bits 存 PTE_COW

理論上可以挪用某些未用 bit，但 RISC-V 規格已明確保留 bits 8–9 給 supervisor software。用它們比較直接：硬體忽略、語意清楚、也不會和未來硬體功能衝突。

另一個選項是用每個行程自己的 bitmap 追蹤 COW 狀態，但這樣需要額外同步，維護成本高。放在 PTE 的 RSW bits 更簡單，而且天然就是 per-PTE 的。

### 2. 用全域陣列做 Reference Counting

```
pageref.count[PHYSTOP / PGSIZE]
```

| 做法 | 複雜度 | 記憶體開銷 | 查找時間 |
|------|--------|------------|----------|
| 全域陣列（採用） | 低 | 固定（小） | O(1) |
| 嵌進 struct run | 中 | 0 | O(1) |
| 額外動態配置 | 高 | 可變 | O(1) |

用 `pa / PGSIZE` 直接索引，O(1)、好理解，開銷也小（`PHYSTOP / PGSIZE` 個整數大概就幾 KB）。代價是只有一把 `pageref.lock`，高平行度下可能有競爭。需要擴充的話可做 per-page lock 或 lock striping，xv6 不需要。

### 3. ref count == 1 的快速路徑

COW fault 發生時，如果該頁只剩目前行程在用，直接略過複製——恢復 `PTE_W`、清 `PTE_COW`，不用配置，不用 copy，O(1)。

真實 workload 很常碰到這種情況：子行程 `exec()` 後會換掉自己的 address space，父行程很多頁面的 ref count 就回到 1 了，下一次寫入就不需要複製。

### 4. uvmcopy() 必須同時更新父子 PTE

`fork()` 時要清掉父行程 PTE 的 `PTE_W`，不是只改子行程。

| 情境 | Parent PTE_W | Child PTE_W | 結果 |
|------|--------------|-------------|------|
| 正確 | 清除 | 清除 | 兩邊寫入都會 fault |
| 錯誤（只改 child） | 保留 | 清除 | parent 可直接寫；child 讀到過期資料 |

如果忘了改 parent，parent 會持續改共享實體頁，child 可能讀到不一致內容。

### 5. Text 頁面的處理

Text 頁面原本就是唯讀映射（`PTE_W = 0, PTE_COW = 0`）。寫入 text 頁面時：

```
pte.valid == 1
pte.PTE_W == 0
pte.PTE_COW == 0   ← 不是 COW
```

fault handler 要把這種情況當成非法寫入並殺掉行程，不是嘗試複製。靠 `PTE_COW` 就能正確區分。

---

## 常見陷阱

### 1. uvmcopy() 忘記更新父行程 PTE

```c
// 錯誤：只把 child 設成 COW
mappages(child_pt, va, PGSIZE, pa, (flags | PTE_COW) & ~PTE_W);
// parent 仍然可寫！
```

```c
// 正確：parent 也要改成 COW
uint64 cow_flags = (flags | PTE_COW) & ~PTE_W;
*pte = PA2PTE(pa) | cow_flags;              // 更新父行程 PTE
mappages(child_pt, va, PGSIZE, pa, cow_flags);  // 再映射 child
```

不改 parent 的話，parent 改共享頁面時 child 會直接看到，靜默資料毀損。

### 2. 還沒 copy 就先減 ref count

```c
// 錯誤：ref count 降下去，舊頁可能被釋放
kfree((void *)old_pa);
memmove(new_page, (void *)old_pa, PGSIZE);  // use-after-free！
```

```c
// 正確：先拷貝，再釋放舊引用
memmove(new_page, (void *)old_pa, PGSIZE);
kfree((void *)old_pa);
```

如果在 `kfree` 和 `memmove` 之間頁面被別人釋放，讀到的是垃圾資料，甚至 panic。

### 3. 更新父行程 PTE 後沒 flush TLB

在 `uvmcopy()` 改完父行程 PTE 後，TLB 可能仍快取舊的可寫映射。xv6 用 `sfence.vma` flush TLB，通常 context switch 時會處理，但要知道：TLB 若留著舊映射，寫入可能繞過保護，直接跳過 COW fault。

### 4. COW fault 時記憶體不足（OOM）

`kalloc()` 失敗時，行程已無法安全繼續，正確做法是 kill process。不要回傳 0 後悄悄繼續，不然 user process 會寫到自己不該擁有的頁面。

```c
void *new_page = kalloc();
if (new_page == 0) {
    p->killed = 1;
    return -1;
}
```

### 5. Reference Count 競態條件

```
Process A reads ref count: 1
Process B reads ref count: 1       ← 兩邊都以為自己是最後一個
Process A skips copy, sets PTE_W
Process B skips copy, sets PTE_W   ← 兩個行程共用同一個「可寫」頁面
```

檢查與更新要包在同一段 `pageref.lock` 保護下，不是只鎖讀取。

### 6. 共享頁面被重複釋放（double free）

```
Process A: count = 1 → 0, free page
Process B: count = 1 → 0, free page  ← double free
```

decrement 與是否 free 的判斷必須在同一把鎖內原子完成。

### 7. uvmcopy() 失敗時的部分清理

`uvmcopy()` 做到一半失敗（例如 `mappages` 跑完記憶體），先前映射的頁面都要回收。ref count 先前已加過，所以清理時用 `do_free = 1` 呼叫 `uvmunmap`，讓它觸發 `kfree` 正確遞減計數。

```c
on_error:
    uvmunmap(child_pagetable, 0, mapped_pages, 1);  // calls kfree → 遞減 ref count
    return -1;
```
