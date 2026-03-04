# xv6 System Calls（系統呼叫）

https://pdos.csail.mit.edu/6.1810/2025/labs/syscall.html

## 目錄
1. [系統呼叫怎麼運作](#系統呼叫怎麼運作)
2. [Lab 任務總覽](#lab-任務總覽)
3. [任務 1：GDB 除錯](#任務-1gdb-除錯)
4. [任務 2：interpose() — 行程沙箱](#任務-2interpose--行程沙箱)
5. [任務 3：記憶體攻擊](#任務-3記憶體攻擊)
6. [Code Sketches](#code-sketches)
7. [關鍵設計決策](#關鍵設計決策)

---

## 系統呼叫怎麼運作

系統呼叫是從 user space 受控地跳進 kernel。xv6 的完整路徑如下：

```text
user code calls write()
  → usys.S stub: 把 syscall 編號放進 a7，執行 ecall
  → hardware: 把 PC 存進 sepc，切到 supervisor mode，跳到 stvec
  → uservec (trampoline): 把所有暫存器存進 trapframe
  → usertrap(): 判斷原因是 syscall（scause == 8）
  → syscall(): 讀 a7，分派到 syscalls[num]()
  → sys_write(): 從 trapframe 的 a0–a5 取參數並執行
  → 回傳值寫回 trapframe->a0
  → userret: 還原暫存器，sret
  → 回到 user code，a0 就是回傳值
```

### Syscall 編號分派

`syscall.c` 用一個簡單的跳表管理 syscall handler：

```c
syscalls[SYS_write]  = sys_write;
syscalls[SYS_read]   = sys_read;
syscalls[SYS_fork]   = sys_fork;
...
```

`a7` 裡放的是 syscall 編號（定義在 `syscall.h`）。dispatcher 查表後呼叫對應函式，回傳值再寫到 `trapframe->a0`，也就是 user 端看到的函式回傳值。

### 參數怎麼傳遞

user space 把參數放在 `a0`–`a5`。kernel 不是直接讀目前暫存器，而是從 **trapframe** 讀（因為已經是不同執行脈絡）：

```text
argint(n, &val)     → 從 trapframe->a0 + n*8 讀整數
argaddr(n, &val)    → 從 trapframe->a0 + n*8 讀位址
argstr(n, buf, max) → 先從 trapframe 讀指標，再從 user space 複製字串
```

`argstr` 是關鍵：它先讀到的是 *user 指標*，接著必須把字串從 **user 位址空間** 複製到 kernel 記憶體，這要用 `copyinstr()`，不能直接解參考。直接在 kernel 解 user pointer 是安全性漏洞。

---

## Lab 任務總覽

| 任務 | 你要做什麼 | 核心觀念 |
|------|------------|----------|
| GDB 除錯 | 回答 trap 狀態相關問題 | trapframe 版面、sepc |
| `interpose()` | 用 bitmask 做 syscall 沙箱 | bitmask 管控、fork 繼承 |
| 記憶體攻擊 | 從釋放頁面中找回 secret | 核心記憶體清零（或未清零） |

---

## 任務 1：GDB 除錯

### 要觀察什麼

在 `syscall()` 設斷點並看狀態，通常會學到：

- `p->trapframe->a7` 是 syscall 編號
- `p->trapframe->epc` 是 `ecall` 指令位址（sepc）
- `syscall()` 結束後，`epc` 會加 4，讓 user code 從 **下一條** 指令續跑

### 常見 GDB 流程

```gdb
(gdb) break syscall
(gdb) continue
(gdb) print p->trapframe->a7     # syscall 編號
(gdb) print/x p->trapframe->epc  # ecall 位址
(gdb) x/i p->trapframe->epc      # 反組譯，應該看到 ecall
```

### Kernel Panic Backtrace

kernel panic 時會呼叫 `backtrace()`（在 Traps lab 實作）。印出的位址可對應到核心函式，常用 `addr2line` 或 `nm kernel/kernel` 來解碼。

---

## 任務 2：interpose() — 行程沙箱

### 目標

新增系統呼叫 `interpose(mask, allowed_path)`，限制呼叫它的行程可用哪些 syscall：

- `mask`：**禁止** syscall 編號的 bitmask（第 `n` bit=1 表示封鎖 syscall `n`）
- `allowed_path`：`open` / `exec` 的例外路徑；若被封鎖的呼叫目標路徑符合這個值，仍允許

### 資料模型

在 `struct proc` 加兩個欄位：

```c
struct proc {
    ...
    int  syscall_mask;       // bitmask: bit n = 1 表示 syscall n 被禁止
    char allowed_path[128];  // open/exec 的例外路徑
}
```

### 檢查點

檢查放在 `syscall()`，分派前：

```c
if ((p->syscall_mask >> num) & 1) {
    if (num == SYS_open || num == SYS_exec) {
        // 在 syscall handler 內處理（需要 path 參數）
        p->trapframe->a0 = syscalls[num]();
    } else {
        p->trapframe->a0 = -1;  // 直接擋下
    }
} else {
    p->trapframe->a0 = syscalls[num]();
}
```

`open` 和 `exec` 特別之處在於：要不要放行跟「操作哪個檔案」有關。這個資訊只有進 handler 解析參數後才知道，dispatch 層拿不到。

### fork 繼承

沙箱設定要跨 `fork()` 繼承。子行程應至少和父行程一樣受限：

```c
// 在 fork() 內
np->syscall_mask = p->syscall_mask;
safestrcpy(np->allowed_path, p->allowed_path, sizeof(np->allowed_path));
```

若沒繼承，被沙箱限制的行程可以 `fork()` 出一個沒限制的 child，直接繞過規則。

---

## 任務 3：記憶體攻擊

### 漏洞來源

xv6 的 `kalloc()` 正常會把新配置頁面清零（`memset`）。若故意移除清零（lab 故意這樣做），被釋放頁面會保留舊內容。下一個行程 `sbrk()` 拿到同一頁時，就能讀到前一個行程留下的資料。

### 攻擊策略

「受害者」行程分配 buffer、寫入敏感資料後結束，把頁面還給 free list。攻擊者行程接著：

1. 用 `sbrk()` 申請一大塊記憶體
2. 掃描取得頁面中的可辨識標記（marker 字串）
3. 讀 marker 旁邊的 secret

```text
受害者 buffer 版面：
  offset 0:  "This may help."   ← 已知 marker
  offset 16: <secret 8 bytes>    ← 目標資料
```

### 為什麼可行

kernel free list 是 LIFO（後進先出，像 stack）。受害者結束時釋放的頁面會在 list 頂端，攻擊者的 `sbrk()` 又從同一個 list 取頁面。若沒有清零，舊資料會原封不動留著。

### 結論

這是典型的 **use-after-free 資訊洩漏** 類型。正確修補是「配置時清零（zero-on-alloc）」，不是「釋放時清零（zero-on-free）」。

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

### 2. 在 syscall() 做封鎖檢查

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
            p->trapframe->a0 = -1;       // 立即拒絕
        } else {
            p->trapframe->a0 = syscalls[num]();  // 繼續呼叫（handler 內可能再檢查 path）
        }
    } else {
        printf("unknown syscall %d\n", num);
        p->trapframe->a0 = -1;
    }
}
```

### 3. 在 sys_open() / sys_exec() 內做 path 例外

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
            // 例外：這個 path 明確允許
        } else {
            return -1;
        }
    }

    // 正常 open 流程
    ...
}
```

### 4. fork() 繼承

```c
int
fork(void)
{
    struct proc *p = myproc();
    struct proc *np;

    // ... 複製記憶體、檔案描述符等 ...

    np->syscall_mask = p->syscall_mask;
    safestrcpy(np->allowed_path, p->allowed_path, sizeof(np->allowed_path));

    ...
}
```

### 5. 記憶體攻擊

```c
void
attack(void)
{
    // 配置頁面，可能拿到受害者剛釋放的頁面
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

## 關鍵設計決策

### 1. Bitmask 表示法

用 bitmask 表示「被禁止的 syscall 編號集合」是最自然的做法：

```c
mask = (1 << SYS_write) | (1 << SYS_open)
```

- O(1) 檢查：`(mask >> num) & 1`
- 放在單一 `int` 就夠（xv6 syscall 約 25 個）
- 好組合：`child_mask = parent_mask | extra_restrictions`

我也考慮過 list/array，但在這個 lab 用 bitmask 更直接、成本更低。

### 2. 為什麼 open / exec 要特別處理

大多數 syscall 不看參數也能在 dispatch 層直接封鎖。但 `open` / `exec` 有 path 例外：即便被封鎖，特定路徑還是允許。

所以這段檢查必須在 syscall 內部做，等讀完 user space 的 path 參數才有資訊判斷。在 dispatch 層做會導致重複解析參數，維護上也容易出錯。

### 3. `allowed_path = "-"` 代表沒有例外

`"-"` 表示「沒有 path 例外，全部擋掉」。這種 sentinel 稍微不直觀，但在 xv6 這個範圍內可接受。若是正式系統，會傾向加一個獨立布林欄位。

### 4. 配置時清零，而非釋放時清零

| 策略 | 何時清零 | 洩漏風險 |
|------|----------|----------|
| free 時清零 | 釋放當下 | 仍可能在某些路徑拿到非零頁面 |
| alloc 時清零 | 配置當下 | 拿到頁面的接收者一定是乾淨頁面 |

核心不變量是：每個 `kalloc()` 呼叫者都應拿到乾淨頁面。只在 free 時清零，仍可能留下舊資料外洩。

