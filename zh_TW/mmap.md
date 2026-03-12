# xv6 Memory-Mapped Files

https://pdos.csail.mit.edu/6.1810/2025/labs/mmap.html

## 目錄
1. [什麼是 Memory-Mapped Files？](#什麼是-memory-mapped-files)
2. [核心概念](#核心概念)
3. [實作架構](#實作架構)
4. [Pseudo Code](#pseudo-code)
5. [設計決策](#設計決策)
6. [常見陷阱](#常見陷阱)

---

## 什麼是 Memory-Mapped Files？

Memory-mapped files 的概念是：把檔案內容映射到行程的虛擬位址空間，之後就能用存取記憶體的方式直接讀寫檔案。

好處主要有幾點：很多情境下可以用 load/store 取代 `read/write`；多個行程可以映射同一個檔案來共享資料；而且只在真的碰到頁面時才載入（lazy loading），讓 `mmap` 能在 O(1) 返回，減少不必要的 user/kernel 切換。

### 基本使用示例

```c
// 開啟檔案
int fd = open("data.txt", O_RDWR);

// 映射檔案到記憶體
void *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, 0);

// 像訪問記憶體一樣訪問檔案
char *data = (char *)addr;
data[0] = 'A';  // 修改會寫回檔案（如果是 MAP_SHARED）

// 解除映射
munmap(addr, 4096);
```

---

## 核心概念

### 1. VMA (Virtual Memory Area)

VMA 用來描述一段連續虛擬記憶體區域：

```c
struct vma {
  int used;              // 此槽位是否使用
  uint64 addr;           // 起始虛擬地址
  uint64 length;         // 長度（字節）
  int prot;              // 權限 (PROT_READ | PROT_WRITE)
  int flags;             // MAP_SHARED 或 MAP_PRIVATE
  struct file *file;     // 映射的檔案
  uint64 offset;         // 檔案偏移
};
```

### 2. Lazy Allocation

呼叫 `mmap` 時不會立刻分配實體記憶體，只建立 VMA 記錄。首次訪問時觸發 page fault，handler 才分配實體頁並載入檔案內容。

這樣做的理由很直接：映射 1GB 的檔案時，`mmap` 可以立即返回，記憶體也只花在真正被訪問到的頁面上。Eager 的做法會讓 `mmap` 變成 O(n) 並且無法處理大檔案。

### 3. MAP_SHARED vs MAP_PRIVATE

| 特性 | MAP_SHARED | MAP_PRIVATE |
|------|------------|-------------|
| 修改可見性 | 寫回檔案 | 不寫回檔案 |
| 用途 | 行程間共享、持久化 | 私有副本、臨時修改 |
| 實現 | munmap/exit 時寫回 | 完全忽略寫回 |

---

## 實作架構

### 系統組件

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

### 主要流程

**mmap 流程:**
```
mmap() → 創建 VMA → 返回虛擬地址
       → 增加檔案參考計數
       → 不分配實體頁
```

**首次訪問流程:**
```
訪問地址 → Page Fault → 檢查 VMA
         → 分配實體頁
         → 從檔案讀取內容 (readi)
         → 建立頁表映射
         → 回到 user space 繼續執行
```

**munmap 流程:**
```
munmap() → 找到對應 VMA
         → 寫回修改（如果 MAP_SHARED）
         → 解除頁表映射 (uvmunmap)
         → 更新/刪除 VMA
         → 減少檔案參考計數
```

---

## Pseudo Code

### 1. mmap 系統呼叫

```c
uint64
sys_mmap(uint64 addr, uint64 length, int prot, int flags, int fd, uint64 offset)
{
    // 驗證參數和檔案權限
    validate_params(length, offset, fd, prot, flags);

    struct proc *p = myproc();

    // 找到空閒 VMA
    struct vma *v = find_free_vma(p);

    // 分配虛擬地址（從 process.sz 向上增長）
    length = PGROUNDUP(length);
    addr = p->sz;
    p->sz += length;

    // 初始化 VMA（不分配實體記憶體）
    v->addr = addr;
    v->length = length;
    v->prot = prot;
    v->flags = flags;
    v->file = filedup(f);  // 重要：增加檔案參考計數

    return addr;
}
```

### 2. Page Fault Handler (vmfault)

```c
void *
vmfault(pagetable_t pagetable, uint64 va, int is_write)
{
    va = PGROUNDDOWN(va);

    // 檢查地址是否在某個 VMA 範圍內
    struct vma *v = find_vma_for_address(myproc(), va);
    if (v == 0)
        return (void *)-1;  // FAILURE

    // 權限檢查（寫入只讀映射會失敗）
    if (is_write && !(v->prot & PROT_WRITE))
        return (void *)-1;

    // 分配並清零實體頁
    void *page = kalloc();
    memset(page, 0, PGSIZE);

    // 從檔案讀取內容
    uint64 offset = v->offset + (va - v->addr);
    struct inode *ip = v->file->ip;
    ilock(ip);
    readi(ip, 0, (uint64)page, offset, PGSIZE);
    iunlock(ip);

    // 建立頁表映射
    int perm = PTE_U | prot_to_pte(v->prot);
    mappages(pagetable, va, PGSIZE, (uint64)page, perm);

    return page;
}
```

### 3. munmap 系統呼叫

```c
int
sys_munmap(uint64 addr, uint64 length)
{
    struct proc *p = myproc();

    // 找到對應的 VMA
    struct vma *v = find_vma_for_address(p, addr);

    // MAP_SHARED：寫回已映射的頁面
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

    // 解除頁表映射並釋放實體頁
    uvmunmap(p->pagetable, addr, length / PGSIZE, 1);

    // 更新或刪除 VMA
    if (addr == v->addr && length == v->length) {
        // 解除整個區域的映射
        v->used = 0;
        fileclose(v->file);
    } else if (addr == v->addr) {
        // 從頭部解除映射
        v->addr += length;
        v->length -= length;
    } else {
        // 從尾部解除映射
        v->length -= length;
    }

    return 0;  // SUCCESS
}
```

### 4. Fork 中的 VMA 處理

```c
int
fork(void)
{
    // ... 創建子行程、複製頁表等 ...

    struct proc *p = myproc();
    struct proc *np = /* newly allocated child proc */;

    // 複製所有 VMA（但不複製實體頁面）
    for (int i = 0; i < NVMA; i++) {
        struct vma *v = &p->vmas[i];
        if (v->used) {
            np->vmas[i] = *v;                        // 複製結構
            np->vmas[i].file = filedup(v->file);     // 增加檔案參考
        }
    }

    // 子行程通過 page fault 載入自己的頁面副本
}
```

父子行程不共享實體頁面，簡化實現並避免 COW 複雜性。

### 5. Exit 中的 VMA 清理

```c
void
exit(int status)
{
    // ... 關閉檔案描述符、當前目錄等 ...

    struct proc *p = myproc();

    // 清理所有 VMA
    for (int i = 0; i < NVMA; i++) {
        struct vma *v = &p->vmas[i];
        if (v->used) {
            // MAP_SHARED：寫回所有已映射的頁
            if (v->flags & MAP_SHARED)
                writeback_all_pages(v);

            // 解除映射並釋放實體頁
            uvmunmap(p->pagetable, v->addr, v->length / PGSIZE, 1);

            // 關閉檔案（減少參考計數）
            fileclose(v->file);
            v->used = 0;
        }
    }
}
```

---

## 設計決策

### 1. 虛擬位址分配：從 `process.sz` 往上長

```
+-------------------+  ← MAXVA
| TRAMPOLINE        |
+-------------------+  ← MAXVA - PGSIZE
| TRAPFRAME         |
+-------------------+  ← MAXVA - 2*PGSIZE
|                   |
|  (未使用空間)     |
|                   |
+-------------------+  ← process.sz (動態增長)
| mmap region 2     |  ← 第二次 mmap
+-------------------+
| mmap region 1     |  ← 第一次 mmap
+-------------------+
|     HEAP          |  ← sbrk() 分配
+-------------------+
|     DATA          |
+-------------------+
|     TEXT          |
+-------------------+  ← 0x0
```

實作簡單，跟 heap 的成長方向一致，不用另外做一套位址管理。代價是會壓縮 heap 可用空間，`mmap` 後再 `sbrk` 容易打架。替代方案是從 TRAPFRAME 往下長，比較複雜但不會卡住 heap。

### 2. VMA 結構：固定大小陣列

```c
#define NVMA 16
struct proc {
    // ...
    struct vma vmas[NVMA];
};
```

16 個槽位，不用動態配置記憶體，遍歷成本可預期，除錯也直覺。限制是同時映射數量被上限卡住，空槽位會浪費一點記憶體。需要彈性的話可用鏈表，但要處理記憶體分配；紅黑樹查找更快，但對 xv6 來說過於複雜。

### 3. Lazy Allocation：在 page fault 才載入

| 方案 | mmap 時間 | 記憶體使用 | 支援大檔案 |
|------|----------|----------|-----------|
| Eager (mmap 時載入) | O(n) | 高 | ✗ |
| Lazy (fault 時載入) | O(1) | 低 | ✓ |

```c
// 映射 1GB 檔案
void *addr = mmap(NULL, 1GB, ...);
// Eager: 需要分配 256K 個頁面（耗時數秒）
// Lazy: 立即返回（幾微秒）

// 只訪問第一頁
char c = ((char*)addr)[0];
// Eager: 已經載入，直接訪問
// Lazy: page fault，只載入 1 頁
```

### 4. MAP_SHARED 的寫回時機：在 `munmap` 和 `exit` 時

每次修改都立即寫回太慢，同一頁也可能被多次修改。在 `munmap`/`exit` 時批量寫回更合理，也不需要追蹤 dirty bit。

目前的實作會寫回所有已映射的頁，不管有沒有被修改——犧牲一點效能換取實作的簡單性。若要最佳化，可以檢查 PTE dirty bit 跳過未修改的頁：

```
if pte & PTE_DIRTY:
    write_back_to_file()
else:
    skip  // 未修改的頁不寫回
```

### 5. Fork 的頁面共享策略：不共享實體頁面

| 方案 | 複雜度 | 記憶體使用 | 一致性 |
|------|--------|----------|--------|
| 共享頁面 (COW) | 高 | 低 | 複雜 |
| 不共享 | 低 | 高 | 簡單 |

```c
// 方案 1: 共享頁面（未採用）
fork() {
    // 標記父行程頁面為只讀
    // 子行程共享相同實體頁
    // 寫入時觸發 COW
}

// 方案 2: 不共享（採用）
fork() {
    // 只複製 VMA 元資料
    // 子行程通過 page fault 載入自己的副本
}
```

---

## 常見陷阱

### 1. 鎖順序造成死鎖

```c
// 錯誤：可能死鎖
ilock(ip);
begin_op();
writei(...);
end_op();
iunlock(ip);
```

```c
// 正確：begin_op 必須在外層
begin_op();
ilock(ip);
writei(...);
iunlock(ip);
end_op();
```

`begin_op` 可能要等 journal 空間；若此時還握著 inode 鎖，就有機會互卡。

### 2. 檔案 reference count 處理錯誤

```c
// 錯誤：未增加參考計數
v->file = f;  // 直接賦值
```

```c
// 正確：使用 filedup
v->file = filedup(f);  // 增加參考計數
```

沒有增加 reference count 的話，使用者 `close(fd)` 後檔案可能先被釋放，VMA 就會留下懸空指標：

```c
int fd = open("file.txt", O_RDWR);
void *addr = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);
close(fd);  // 如果沒有 filedup，這裡會釋放檔案

// 訪問 addr 會崩潰（懸空指標）
char c = ((char*)addr)[0];  // 崩潰或讀取垃圾資料
```

### 3. 權限檢查時機不對

```c
// 錯誤：分配後才檢查
mem = kalloc();
if (!(vma->prot & PROT_WRITE))
    return 0;  // 洩露了 mem！
```

```c
// 正確：分配前檢查
if (!write && !(vma->prot & PROT_WRITE))
    return 0;
mem = kalloc();
```

### 4. 沒處理檔案尾端的不完整頁面

檔案大小不一定是頁面大小的整數倍。

```c
// 錯誤：可能寫超出檔案
writei(ip, 0, pa, file_offset, PGSIZE);
```

```c
// 正確：限制寫入大小
max_write = PGSIZE;
if (file_offset + max_write > ip->size)
    max_write = ip->size - file_offset;

if (max_write > 0)
    writei(ip, 0, pa, file_offset, max_write);
```

```
檔案大小: 4100 bytes
映射大小: 8192 bytes (2 pages)

Page 0: [0, 4096)     → 完整頁，寫回 4096 bytes ✓
Page 1: [4096, 8192)  → 部頁面，只寫回 4 bytes ✓
```

### 5. 忘了更新 `process.sz`

```c
// 錯誤：未擴展地址空間
addr = p->sz;
v->addr = addr;
v->length = length;
// 沒有增加 p->sz
```

```c
// 正確：擴展地址空間
addr = p->sz;
p->sz += length;  // 必須更新
v->addr = addr;
v->length = length;
```

之後 `sbrk` 或新的 `mmap` 可能直接蓋到這段映射區域。

### 6. 忘記清零實體頁

```c
// 錯誤：未清零
mem = kalloc();
readi(ip, 0, mem, offset, PGSIZE);
```

```c
// 正確：先清零
mem = kalloc();
memset(mem, 0, PGSIZE);  // 必須清零
readi(ip, 0, mem, offset, PGSIZE);
```

`kalloc` 返回的頁面可能包含之前的資料。如果檔案小於一頁，`readi` 只填充部分內容，未讀部分應該為 0（Unix 標準行為）。

---

## 效能最佳化建議

xv6 優先考慮簡單性，以下是可能的最佳化方向：

### 1. Dirty Bit 追蹤

目前寫回所有已映射的頁。改成只寫回修改過的頁可以減少不必要的磁盤 I/O：

```c
// 檢查 PTE dirty bit
if (pte & PTE_D) {  // PTE_D = dirty bit
    writei(...);
    *pte &= ~PTE_D;  // 清除 dirty bit
}
```

### 2. 批量寫回

目前每頁一個事務（`begin_op/end_op`）。批量處理多頁可以減少日誌事務開銷：

```c
begin_op();
ilock(ip);
for each dirty page:
    writei(...);
iunlock(ip);
end_op();
```

### 3. VMA 合併

相鄰的同類映射可以自動合併，減少 VMA 使用：

```c
// 檢查是否可以與現有 VMA 合併
for each vma:
    if vma.addr + vma.length == new_addr and
       vma.file == new_file and
       vma.prot == new_prot and
       vma.flags == new_flags:
        // 合併而非創建新 VMA
        vma.length += new_length
        return vma.addr
```

### 4. 預讀 (Readahead)

目前只載入觸發 page fault 的頁。預測性載入相鄰頁面可以減少 fault 次數：

```c
vmfault(va) {
    // 載入當前頁
    load_page(va);

    // 預讀下幾頁
    if (sequential_access_pattern) {
        for i = 1 to READAHEAD_SIZE:
            load_page(va + i * PGSIZE);
    }
}
```

### 手動測試案例

```c
// 測試 1: 基本功能
int fd = open("README", O_RDONLY);
char *p = mmap(0, 100, PROT_READ, MAP_PRIVATE, fd, 0);
printf("%c", p[0]);  // 應該成功
munmap(p, 100);

// 測試 2: 寫入只讀映射（應該失敗）
p = mmap(0, 100, PROT_READ, MAP_PRIVATE, fd, 0);
p[0] = 'X';  // 應該觸發 page fault → 行程終止

// 測試 3: 檔案關閉後仍可訪問
int fd = open("file.txt", O_RDWR);
char *p = mmap(0, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
close(fd);  // 關閉檔案描述符
p[0] = 'A';  // 仍然可以訪問（因為 filedup）
munmap(p, 4096);

// 測試 4: MAP_SHARED 寫回
fd = open("output.txt", O_RDWR);
p = mmap(0, 100, PROT_WRITE, MAP_SHARED, fd, 0);
p[0] = 'H';
p[1] = 'i';
munmap(p, 100);  // 寫回檔案
// 重新讀取檔案，應該看到 "Hi"
```
