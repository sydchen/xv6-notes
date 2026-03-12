# xv6 Traps

https://pdos.csail.mit.edu/6.1810/2025/labs/traps.html

## 目錄
1. [什麼是 Trap？](#什麼是-trap)
2. [RISC-V Trap 機制](#risc-v-trap-機制)
3. [Lab 任務總覽](#lab-任務總覽)
4. [任務 1：Backtrace](#任務-1backtrace)
5. [任務 2：Alarm](#任務-2alarm)
6. [Code Sketches](#code-sketches)
7. [設計決策](#設計決策)
8. [常見陷阱](#常見陷阱)

---

## 什麼是 Trap？

**Trap** 是控制流從 user space 轉進 kernel space。以 RISC-V（和 xv6）來說，常見三種來源：

| 事件 | 原因 | 例子 |
|------|------|------|
| **System call** | `ecall` 指令 | `write()`, `fork()` |
| **Exception** | 非法指令或非法記憶體存取 | Page fault、除以零 |
| **Device interrupt** | 硬體中斷 CPU | timer tick、磁碟 I/O 完成 |

三者都走同一條 trap handling 路徑：硬體先儲存最小狀態、跳進核心 trap handler，核心判斷發生了什麼，最後再回到 user space。

---

## RISC-V Trap 機制

### 硬體負責的事（CPU 自動完成）

當 trap 發生時，RISC-V 硬體會：
1. 把目前 PC 存到 `sepc`（supervisor exception PC）
2. 把原因存到 `scause`
3. 將特權等級切到 supervisor
4. 跳到 `stvec`（supervisor trap vector）指定的位置

硬體其實只幫你存很少——暫存器、stack pointer 等其餘狀態都要核心自己儲存。這就是每個行程都有獨立 trapframe 的原因。

### Trapframe

trapframe 是每個行程一份的核心資料結構，記錄 trap 當下所有 user registers 的快照：

```c
struct trapframe {
    uint64 kernel_satp;   // kernel page table
    uint64 kernel_sp;     // kernel stack pointer
    uint64 kernel_trap;   // usertrap() 的位址
    uint64 epc;           // 儲存的 user program counter  ← sepc
    uint64 kernel_hartid; // CPU core ID
    uint64 ra, sp, gp, tp;
    uint64 t0 ... t6;
    uint64 s0 ... s11;
    uint64 a0 ... a7;     // a0-a7: syscall 參數 / 回傳值
}
```

回到 user space 時，核心會：
1. 從 trapframe 還原所有暫存器
2. 把 `sepc` 設為 `trapframe->epc`（使用者程式要續跑的位置）
3. 執行 `sret`（supervisor return）

### RISC-V Stack Frame 版面

每次函式呼叫都會在 stack 上建立一個 stack frame：

```text
高位址
+------------------+
|  return address  |  ← fp - 8   （函式返回位置）
+------------------+
|  saved fp (s0)   |  ← fp - 16  （上一層 frame 的 fp）
+------------------+
|  local variables |
|  ...             |
+------------------+  ← sp（stack pointer，向下成長）
低位址
```

frame pointer（s0/fp）會指向目前 frame 頂端。沿著「saved fp 鏈」往回追，就能走完整個 call stack，這就是 `backtrace()` 的核心。

---

## Lab 任務總覽

這個 lab 有兩個獨立任務：

1. Backtrace：走訪 kernel call stack 並印出 return address（除錯工具）
2. Alarm：實作 `sigalarm(interval, handler)` / `sigreturn()`，定期把控制流導向 user-space handler

---

## 任務 1：Backtrace

### 目標

在核心中實作 `backtrace()`，印出目前 kernel stack 上每一層 frame 的 return address。`panic()` 時會自動呼叫。

### Stack Walking 怎麼做

```text
目前 frame:
  fp → [return addr | saved fp | locals...]
           ↓              ↓
         印這個       跳到這裡 → 重複
```

從目前 `s0`（frame pointer）出發：
1. 讀 `fp - 8` 的 return address 並印出
2. 讀 `fp - 16` 的上一層 frame pointer
3. 重複直到離開當前頁面（xv6 每個 kernel stack 正好一頁）

### 什麼時候停止

xv6 給每個 kernel stack 恰好一頁。當 frame pointer 超出這一頁邊界，就代表你已經走出這個 stack 範圍。

---

## 任務 2：Alarm

### 目標

實作兩個新 syscall：

- `sigalarm(n, handler)`：每 `n` 個 CPU ticks 在 user space 呼叫 `handler()`
- `sigreturn()`：由 handler 呼叫，用來恢復被打斷的原本執行點

### 難點：上下文儲存

棘手之處在於 `handler()` 會插入在使用者程式執行的「中間」。handler 結束（透過 `sigreturn`）後，原程式必須像「沒發生任何事」一樣繼續跑：暫存器、PC、回傳值都要一致。

這表示你必須儲存並還原整份 trapframe。

### 流程總覽

```text
使用者程式正常執行
  → timer interrupt（每 N ticks）
  → kernel 累加 alarm_ticks
  → alarm_ticks == alarm_interval?
      → 將完整 trapframe 存到 alarm_trapframe
      → 將 trapframe->epc 改成 alarm_handler 位址
      → 回 user space（但這次會跑 handler）
  → handler 執行
  → handler 呼叫 sigreturn()
      → 從 alarm_trapframe 還原 trapframe
      → 清掉 in-handler flag
      → 回 user space（續跑原本程式）
```

### 防重入（Re-entrancy）

如果 handler 執行超過 `n` ticks，下一次 timer 可能又打進 handler，造成遞迴中斷甚至 stack 爆掉。

解法：加 `alarm_in_handler` 旗標。這個旗標為真時，就算 tick 到了也不再觸發 alarm。

---

## Code Sketches

以下是簡化版本，用來說明邏輯；語意對齊 xv6，但不是可直接貼上的 C code。

### 1. 讀取 Frame Pointer

```c
// 透過 inline assembly 讀取 s0（frame pointer）
static inline uint64 r_fp() {
    uint64 x;
    asm volatile("mv %0, s0" : "=r"(x));
    return x;
}
```

RISC-V calling convention 會把目前 frame pointer 放在 `s0`，所以 `s0` 是 stack walking 的穩定起點。

### 2. backtrace()

```c
void backtrace(void) {
    uint64 fp = r_fp();
    uint64 page_start = PGROUNDDOWN(fp);

    printf("backtrace:\n");

    while (fp >= page_start && fp < page_start + PGSIZE) {
        uint64 ra = *(uint64 *)(fp - 8);   // return address
        printf("%p\n", (void *)ra);
        fp = *(uint64 *)(fp - 16);          // 上一層 frame pointer
    }
}
```

xv6 每個 kernel stack 固定是 `PGSIZE`，所以用頁邊界當停止條件。

### 3. proc 結構中的 Alarm 欄位

```c
struct proc {
    // ... existing fields ...
    int alarm_interval;               // 幾個 ticks 觸發一次
    void (*alarm_handler)();          // user-space handler 位址
    int alarm_ticks;                  // 距離上次 alarm 的累積 ticks
    struct trapframe alarm_trapframe; // 儲存的完整暫存器快照
    int alarm_in_handler;             // 防重入旗標
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

### 5. usertrap() 中的 timer interrupt

```c
// 在 usertrap() 偵測到 which_dev == 2（timer）之後
if (p->alarm_interval > 0 && !p->alarm_in_handler) {
    p->alarm_ticks++;
    if (p->alarm_ticks >= p->alarm_interval) {
        p->alarm_ticks = 0;
        // 儲存完整暫存器狀態
        memmove(&p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
        // 防止重入
        p->alarm_in_handler = 1;
        // 回 user mode 後改跑 handler，不是直接呼叫
        p->trapframe->epc = (uint64)p->alarm_handler;
    }
}
yield();
```

核心不會直接呼叫 handler，而是改寫 `trapframe->epc`。等 `sret` 後，CPU 在 user mode 直接從 handler 位址繼續。

### 6. sys_sigreturn()

```c
uint64 sys_sigreturn(void) {
    struct proc *p = myproc();

    // 還原 handler 之前的完整暫存器快照
    memmove(p->trapframe, &p->alarm_trapframe, sizeof(struct trapframe));

    // 清除防重入旗標
    p->alarm_in_handler = 0;

    // 回傳儲存的 a0，讓 syscall 機制寫回後不破壞原值
    return p->alarm_trapframe.a0;
}
```

---

## 設計決策

### 1. 儲存整份 Trapframe（不只 epc）

第一直覺可能是只存 `epc`，但 handler 是一般 C 函式，會改動 `a*`、`t*` 等暫存器。正確做法是觸發 alarm 前把整份 trapframe 存到 `alarm_trapframe`，`sigreturn` 再整份還原。

```text
Alarm 前:                 sigreturn 後:
  a0 = 42                   a0 = 42
  t1 = 0xdeadbeef           t1 = 0xdeadbeef
  epc = 0x1234              epc = 0x1234
```

### 2. 用 epc 重新導向，不直接呼叫 handler

核心不能直接呼叫 user-space function pointer，因為位址空間和特權等級不同。

做法是操作 trap return 路徑：
- 把 `trapframe->epc` 改成 handler 位址
- `sret` 後在 user mode 跳到該位址

實務上等於重用同一條 trap-return 流程，只是換了 resume PC。

### 3. `sigreturn` 的 `a0` 覆寫問題

`sigreturn` 是 syscall，而 xv6 syscall 框架會把 syscall 回傳值寫到 `trapframe->a0`：

```c
trapframe->a0 = syscall_return_value
```

如果 `sigreturn` 回傳 `0`，就會把剛恢復好的 `a0` 蓋掉。

做法是讓 `sys_sigreturn()` 回傳儲存的 `a0`：

```c
return p->alarm_trapframe.a0;  // syscall 再寫回同值，等於不變
```

### 4. 防重入旗標（`alarm_in_handler`）

| 情境 | 無 Guard | 有 Guard |
|------|----------|----------|
| Handler < N ticks | 正常 | 正常 |
| Handler > N ticks | 在 handler 裡再次觸發 alarm | 跳過直到 handler 結束 |
| Handler 中途 `sigreturn` | 行為不明 | `sigreturn` 後重新啟用 |

`alarm_in_handler` 應在 alarm 觸發時設為 true，並在 `sigreturn` 清除。

### 5. `alarm_ticks` 何時歸零

`alarm_ticks = 0` 應在 **alarm 觸發當下** 重設，而不是在 `sigreturn`。這樣下一次間隔從觸發點開始算，而非 handler 結束點。

---

## 常見陷阱

### 1. 沒有在進 handler 前儲存全部暫存器

錯誤：
```c
p->alarm_trapframe.epc = p->trapframe->epc;  // 只存 PC
p->trapframe->epc = (uint64)p->alarm_handler;
```

正確：
```c
memmove(&p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));  // 全存
p->trapframe->epc = (uint64)p->alarm_handler;
```

handler 改到的任何暫存器（含編譯器暫存器）都可能讓 user 程式恢復後壞掉。

### 2. 忘記 `sigreturn` 的 `a0` 問題

錯誤：
```c
uint64 sys_sigreturn(void) {
    memmove(p->trapframe, &p->alarm_trapframe, sizeof(struct trapframe));
    p->alarm_in_handler = 0;
    return 0;  // 會把 trapframe->a0 蓋成 0
}
```

正確：
```c
uint64 sys_sigreturn(void) {
    memmove(p->trapframe, &p->alarm_trapframe, sizeof(struct trapframe));
    p->alarm_in_handler = 0;
    return p->alarm_trapframe.a0;
}
```

### 3. 重新導向前沒有先設 Guard

錯誤：
```c
p->trapframe->epc = (uint64)p->alarm_handler;  // 先導向
p->alarm_in_handler = 1;                        // 才設旗標
```

正確：
```c
p->alarm_in_handler = 1;                        // 先設旗標
p->trapframe->epc = (uint64)p->alarm_handler;
```

在真實系統中，先後順序可能影響是否出現競態；xv6 雖簡化，仍建議維持安全順序。

### 4. Backtrace 停止條件錯誤

錯誤：
```c
while (fp != 0) {  // 可能走進其他記憶體
    ...
}
```

正確：
```c
uint64 page_start = PGROUNDDOWN(fp);
while (fp >= page_start && fp < page_start + PGSIZE) {
    ...
}
```

xv6 kernel stack 一頁固定，用頁邊界判斷最安全。

### 5. Stack frame offset 用錯

RISC-V ABI 定義很明確：
- return address：`fp - 8`
- saved frame pointer：`fp - 16`

這是 calling convention 規定，不要自己猜 offset。

