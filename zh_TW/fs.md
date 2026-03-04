# xv6 File System（檔案系統）

https://pdos.csail.mit.edu/6.1810/2025/labs/fs.html

## 目錄
1. [xv6 檔案系統概觀](#xv6-檔案系統概觀)
2. [Lab 任務總覽](#lab-任務總覽)
3. [任務 1：大型檔案（雙重間接區塊）](#任務-1大型檔案雙重間接區塊)
4. [任務 2：Symbolic Links](#任務-2symbolic-links)
5. [Code Sketches](#code-sketches)
6. [關鍵設計決策](#關鍵設計決策)
7. [常見陷阱](#常見陷阱)

---

## xv6 檔案系統概觀

### 磁碟版面（On-Disk Layout）

xv6 把磁碟切成固定大小的區塊（每塊 1024 bytes）：

```text
| boot | super | log | inodes | bitmap | data blocks ... |
```

- **Superblock**：檔案系統中繼資料（大小、inode 數量等）
- **Log**：write-ahead log，用於當機復原
- **Inode blocks**：磁碟上 inode 陣列（`struct dinode`）
- **Bitmap**：每個 data block 一個 bit，追蹤已用/未用
- **Data blocks**：實際檔案與目錄內容

### Inode 與區塊映射

每個檔案由一個 **inode** 表示，裡面保存中繼資料和 data block 位址清單。問題在於：inode 在磁碟上大小固定，無法存無限多個位址。xv6 透過 **indirect blocks** 解這件事：

```text
本 lab 前（原始）：
  addrs[0..10]   → 11 個 direct data blocks
  addrs[11]      → singly-indirect block
                    └─ 256 個 block addresses → 再指到 256 個 data blocks
  上限：11 + 256 = 267 blocks ≈ 267 KB

本 lab 後：
  addrs[0..10]   → 11 個 direct data blocks
  addrs[11]      → singly-indirect block → 256 個 data blocks
  addrs[12]      → doubly-indirect block
                    └─ 256 個第一層指標
                        └─ 每個再指到 256 個 data blocks
  上限：11 + 256 + 256×256 = 65,803 blocks ≈ 64 MB
```

### `bmap()` 函式

`bmap(ip, bn)` 會把檔案中的 **邏輯區塊號**（0, 1, 2, …）轉成 **磁碟區塊號**。若區塊尚不存在，它也會按需配置。

`bmap()` 幾乎是唯一需要真正理解 indirect 版面的地方；多數讀寫路徑都只是依賴它。

---

## Lab 任務總覽

| 任務 | 變更內容 | 核心觀念 |
|------|----------|----------|
| 大型檔案 | 在 `bmap()`、`itrunc()` 加入 doubly-indirect | 兩層索引計算 |
| Symbolic links | 新增 `symlink()` syscall + 修改 `sys_open()` | 路徑遍歷中的 inode 鎖定協定 |

這兩個任務大致上彼此獨立。

---

## 任務 1：大型檔案（雙重間接區塊）

### 問題點

原本 `addrs` 陣列：`addrs[NDIRECT+1]`，其中 `NDIRECT=12`。
- `addrs[0..11]`：12 個 direct blocks
- `addrs[12]`：singly-indirect block

要加入 doubly-indirect，且 **不改動磁碟 inode 大小**，做法是用一個 direct slot 去交換新層級：
- `NDIRECT` 從 12 改成 11
- `addrs[11]`：singly-indirect（角色不變，只是索引位移）
- `addrs[12]`：doubly-indirect（新）
- `addrs` 陣列大小改成 `NDIRECT+2` = 13（總 bytes 與原本相同，因為 NDIRECT 減 1）

### 兩層索引運算

當邏輯區塊號 `bn` 落在 doubly-indirect 範圍時：

```text
bn -= NINDIRECT          // 先扣掉 singly-indirect 範圍

idx1 = bn / NINDIRECT    // 第一層 slot（0..255）
idx2 = bn % NINDIRECT    // 第二層 slot（0..255）

磁碟版面：
  addrs[NDIRECT+1]       → doubly-indirect root block
    [idx1]               → 第一層 indirect block
      [idx2]             → 實際 data block
```

`NINDIRECT = BSIZE / sizeof(uint) = 1024 / 4 = 256`

### itrunc() 釋放順序

truncate 檔案時，區塊必須由內而外釋放：先 data blocks，再釋放指向它們的 blocks。

```text
對 doubly-indirect block：
  1. 走訪每個第一層指標 [j]：
       a. 讀第一層 block
       b. 走訪第二層指標 [k]，釋放 data block
       c. 釋放第一層 block 本身
  2. 釋放 doubly-indirect root block
  3. 設 addrs[NDIRECT+1] = 0
```

若先釋放 root，再去找 child，child 指標就遺失，會造成洩漏。

---

## 任務 2：Symbolic Links

### Symlink 是什麼

symbolic link 是一種檔案，其內容是指向另一個檔案的 **path 字串**。它和 hard link 不同：hard link 直接共享 inode；symlink 自己有獨立 inode，型別是 `T_SYMLINK`。

```text
hard link:    direntry "a" ──► inode 17 (data)
              direntry "b" ──► inode 17 (同一個 inode)

symlink:      direntry "link" ──► inode 42 (T_SYMLINK, content="/path/to/target")
              direntry "target" ──► inode 17 (data)
```

### 新增項目

```c
// stat.h
#define T_SYMLINK 4

// fcntl.h
#define O_NOFOLLOW 0x800   // 開 symlink 本身，不追蹤
```

### sys_symlink()

建立 symlink 的流程：
1. 在 `path` 建一個型別為 `T_SYMLINK` 的 inode
2. 把 `target` 字串寫進它的第一個 data block
3. 完成（沒有特殊格式，就是原始 path 字串）

### 在 sys_open() 追蹤 symlink

當 `open()` 遇到 `T_SYMLINK` inode：
1. 從 inode data 讀出 target path
2. 釋放目前 inode（`iunlockput`）
3. 用 `namei()` 解析新 path
4. 鎖住新 inode
5. 若新 inode 也是 symlink，重複流程（遞迴解析）
6. 深度達到 `>= 10` 就停止（循環偵測）

實務規則：**讀 `ip->type` 時要持有 inode lock**，**呼叫 `namei()` 前一定要先釋放 lock**。

---

## Code Sketches

### 1. 修改 inode 結構

```c
// Before
#define NDIRECT 12
uint addrs[NDIRECT+1];   // 13 entries: 12 direct + 1 singly-indirect

// After
#define NDIRECT 11
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT*NINDIRECT)
uint addrs[NDIRECT+2];   // 13 entries: 11 direct + 1 singly + 1 doubly
//          ↑ 總 entry 數仍是 13，磁碟 inode 大小不變
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

### 3. itrunc() — 釋放 doubly-indirect blocks

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

### 5. sys_open() — follow symlink

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

## 關鍵設計決策

### 1. 用一個 direct slot 換 doubly-indirect

磁碟上的 inode 大小固定，不能直接新增位址欄位，否則會破壞相容性。

實務上最可行的是 size-neutral 交換：
- 減少一個 direct slot（NDIRECT: 12 → 11）
- 在同一空間加入 doubly-indirect slot
- 陣列仍是 13 個 entries（`NDIRECT+2 = 13`，總 bytes 不變）

取捨是：小檔案少一個 direct slot；換來的是更大的最大檔案上限。

### 2. bmap() 按需配置

`bmap()` 不只是 lookup，也會按需配置區塊。所以它必須在 transaction（`begin_op`/`end_op`）內執行，且在修改 metadata 後要呼叫 `log_write()`。

doubly-indirect case 的順序重點：
1. root block 不存在就配置 → `log_write` parent inode
2. 第一層 block 不存在就配置 → `log_write` root block
3. data block 不存在就配置 → `log_write` 第一層 block

任一層漏記錄，當機後都可能留下「只連一半」的 metadata。

### 3. Symlink 內容只存原始 path 字串

symlink 的 target 就是存成一般字串（不做特殊編碼、沒有額外 metadata）。

對 xv6 夠簡單也夠用。缺點是 target path 改名或刪除後，symlink 會變成 dangling。

### 4. Symlink follow 的 inode lock 協定

重要限制：**持有 inode lock 時不要呼叫 `namei()`**。`namei()` 內部會再取 lock，容易死鎖。

每一跳建議順序：

```text
read ip->type    (持有 ilock)
read target path (持有 ilock)
iunlockput(ip)   (先釋放再 namei)
ip = namei(...)  (不持鎖)
ilock(ip)        (鎖新 inode)
```

這個順序一旦弄錯，死鎖風險非常高。

### 5. 用深度上限做循環偵測

若要精準抓 symlink 迴圈（例如 A → B → A），需要追蹤 visited set。xv6 採用簡化版：深度上限 10。

若深度到 10 仍是 symlink，就當作循環並回傳錯誤。這也符合 Linux 的歷史做法（早期 `MAXSYMLINKS = 8`，現在是 40）。

---

## 常見陷阱

### 1. bmap() 忘記扣掉前面層級範圍

每層 indirect 在算本地索引前，都要先扣掉先前層級的 block 數。

**錯誤：**
```c
// Doubly-indirect case
uint idx1 = bn / NINDIRECT;   // bn 尚未扣除 singly-indirect 範圍
```

**正確：**
```c
bn -= NDIRECT;
if (bn < NINDIRECT) { /* singly-indirect */ }
bn -= NINDIRECT;              // 扣掉 singly-indirect 範圍
// 這時 bn 才是 doubly-indirect 的相對索引
uint idx1 = bn / NINDIRECT;
```

### 2. itrunc() 釋放順序錯誤

**錯誤：** 先 free root，再去讀 child 指標。

```c
bfree(ip->dev, ip->addrs[NDIRECT+1]);  // root 已釋放
bp = bread(ip->dev, ip->addrs[NDIRECT+1]);  // 讀已釋放區塊，資料不可信
```

**正確：** 一律先 free leaves，再 free parents。

### 3. 修改 indirect block 後沒呼叫 log_write()

`bmap()` 會改到 indirect block 內容（指標陣列），這些變更必須記錄：

```c
a[idx1] = addr;
log_write(bp);   // 改完 bp->data 後要立刻記錄
```

若沒有 `log_write`，變更只在 buffer cache。當機時還沒落盤，就會留下 0 entry，造成資料遺失。

### 4. 持有 inode lock 時呼叫 namei()

```c
// 錯誤：namei 內部會取 lock，可能死鎖
ilock(ip);
// ... read symlink target ...
ip2 = namei(target);
```

```c
// 正確：先釋放再解析
iunlockput(ip);
ip2 = namei(target);
ilock(ip2);
```

### 5. 忘記處理 O_NOFOLLOW

當 `O_NOFOLLOW` 有設時，`open()` 應回傳 symlink inode 本身，而不是 target。若漏掉這個判斷，symlink 就無法被直接打開（例如讀內容、移除等工具行為會受影響）。

```c
if (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
    // follow the link
}
// 若 O_NOFOLLOW 有設，應直接走一般 open 流程
```

### 6. 沒有深度上限導致無限迴圈

若沒有深度限制，`A → B → A` 會讓 `sys_open()` 無限循環，行程卡死（或最終 stack overflow）。

每跳都要檢查深度，達上限就回傳錯誤。

---

## 各部分怎麼串起來

```text
write(fd, buf, n)
  → writei(ip, buf, offset, n)
      → bmap(ip, block_number)        ← 邏輯區塊號轉磁碟區塊號
          if direct: ip->addrs[bn]
          if singly-indirect: bread(addrs[NDIRECT]) → a[bn]
          if doubly-indirect: bread(addrs[NDIRECT+1])
                               → a[idx1] → bread(a[idx1]) → a[idx2]
      → bread / bwrite data block

open("link", 0)
  → namei("link") → inode 42 (T_SYMLINK)
  → ilock(42), read target="/real/file"
  → iunlockput(42)
  → namei("/real/file") → inode 17
  → ilock(17), continue normal open
```

`bmap()` 是這個 lab 最核心的翻譯層。只要這裡邏輯正確，後續 file growth 與區塊配置行為就會好推理很多。
