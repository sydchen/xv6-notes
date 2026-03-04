# xv6 Networking

https://pdos.csail.mit.edu/6.1810/2025/labs/net.html

## 目錄
1. [背景：網路堆疊分層](#背景網路堆疊分層)
2. [Lab 任務總覽](#lab-任務總覽)
3. [Part 1: E1000 NIC Driver](#part-1-e1000-nic-driver)
4. [Part 2: UDP Receive Stack](#part-2-udp-receive-stack)
5. [Code Sketches](#code-sketches)
6. [關鍵設計決策](#關鍵設計決策)
7. [常見陷阱](#常見陷阱)

---

## 背景：網路堆疊分層

封包在送出與接收路徑都會經過多層：

```text
Send path (user → wire):
  user data
    → UDP header (src/dst port, length)
    → IP header  (src/dst IP, protocol=UDP)
    → Ethernet header (src/dst MAC, type=IPv4)
    → E1000 TX ring → wire

Receive path (wire → user):
  wire → E1000 RX ring
    → e1000_recv() → net_rx()
    → ip_rx()    (filter, find bound port)
    → portqueue  (sleep/wakeup)
    → sys_recv() (copy payload to user)
```

header 是往前 prepend，所以 Ethernet header 永遠在 buffer 最前面。解析時要用正確 offset 做 cast。

### E1000 NIC 與 Descriptor Rings

E1000 是硬體 NIC（在 QEMU 中被模擬）。kernel 透過記憶體中的 **descriptor rings** 和 NIC 溝通：這是 CPU 與 NIC 共用的環狀 descriptor 陣列。

```text
TX ring (transmit):
  CPU writes descriptor → sets TDT → NIC reads and sends → sets DD bit

RX ring (receive):
  NIC writes descriptor + data → sets DD bit → CPU reads → sets RDT
```

`TDT`（Tail Descriptor Transmit）與 `RDT`（Tail Descriptor Receive）是 **NIC register**，標記 producer 最新寫到哪。NIC 會從自己的 head 處理到 tail（不含 tail）。

---

## Lab 任務總覽

| Task | Function | Role |
|------|----------|------|
| TX driver | `e1000_transmit()` | 把封包放進 TX ring 並通知 NIC |
| RX driver | `e1000_recv()` | 清空 RX ring 並往上交給 network stack |
| UDP bind | `sys_bind()` | 註冊行程在某個 port 收封包 |
| UDP recv | `sys_recv()` | 阻塞等待封包並回傳 payload |
| IP filter | `ip_rx()` | 將進來封包匹配到綁定 port 並入列 |

這五個部分會串成一條完整的收送資料路徑。

---

## Part 1: E1000 NIC Driver

### Transmit: e1000_transmit()

TX ring 有 `TX_RING_SIZE`（16）個 descriptor slot。每個 slot 含：
- `addr`：封包 buffer 的實體位址
- `length`：封包長度（bytes）
- `cmd`：命令旗標（`EOP`=封包結尾，`RS`=要求狀態）
- `status`：NIC 完成後設定（`DD`=descriptor done）

協定流程：

```text
1. 讀 TDT — 這是下一個要用的 slot
2. 檢查 tx_ring[TDT].status 的 DD bit
   - 若 DD 為 0：NIC 還沒處理完這格的前一包 → ring full，回傳 -1
3. 釋放這格舊 buffer（前一次傳送留下）
4. 填新 descriptor：addr=新 buffer, length, cmd=EOP|RS
5. 前進 TDT = (TDT + 1) % TX_RING_SIZE
6. NIC 會開始送，完成後把 DD 設回 1
```

network stack 傳入的是 `kalloc()` 出來的 buffer。成功入列後 所有權 轉給 driver，等這格下次被重用時再由 driver `kfree()`。

### Receive: e1000_recv()

RX ring 的語意不同：`RDT` 指向的是「CPU 最後還給 NIC 的 descriptor」，不是下一個要處理的。所以下一格是 `(RDT + 1) % RX_RING_SIZE`。

```text
while (1):
    next = (RDT + 1) % RX_RING_SIZE
    if DD bit at rx_ring[next] is clear: break   // 沒有更多封包

    buf = rx_ring[next].addr                     // 封包 buffer
    len = rx_ring[next].length
    net_rx(buf, len)                              // 往上層傳

    rx_ring[next].addr = kalloc()                // 補一個新 buffer 給 NIC
    rx_ring[next].status = 0                     // 清 DD
    RDT = next                                   // 告知 NIC 此格可再用
```

`e1000_recv()` 跑在 interrupt context，應一次把當前可讀 RX descriptors 都清完。

---

## Part 2: UDP Receive Stack

### Data Structures

```c
#define NPORT   16    // 同時可綁定的 port 上限
#define NPACKET 16    // 每個 port 可排隊封包上限

struct packet {
    char   *buf;       // kalloc 出來的 buffer（完整 Ethernet frame）
    int     len;
    uint32  src_ip;
    uint16  src_port;
};

struct portqueue {
    struct spinlock lock;
    int    bound;
    uint16 port;
    struct packet packets[NPACKET];  // circular buffer
    uint   head;    // 下一個 dequeue
    uint   tail;    // 下一個 enqueue
};

static struct portqueue ports[NPORT];
static struct spinlock  netlock;     // 保護 ports[] 陣列本身
```

這裡有兩層鎖：
- `netlock`：粗粒度，拿來搜尋/修改 `ports[]`
- `pq->lock`：每個 portqueue 一把，保護該 port 的封包佇列（不同 port 可並行 recv）

### sys_bind()

把 port 號綁到某個空 `portqueue` slot：

```text
1. 驗證 port（1–65535）
2. acquire netlock
3. 掃 ports[]：若該 port 已綁，回傳 0（idempotent）
4. 找第一個未綁 slot
5. 設 slot.bound=1, slot.port=port，重設 head/tail
6. release netlock
```

### ip_rx() — 進來封包分派器

`net_rx()` 做完基本 Ethernet/IP parsing 後會呼叫：

```text
1. 檢查 ip->ip_p == IPPROTO_UDP；否則 kfree(buf) 並 return
2. 解析 UDP header：dport = ntohs(udp->dport)
3. 在 ports[] 找對應 bound port
4. 找不到就 kfree(buf) 丟棄
5. acquire pq->lock
6. 若 queue 滿（tail - head >= NPACKET）：kfree(buf) 丟棄
7. enqueue: packets[tail % NPACKET] = {buf, len, src_ip, src_port}; tail++
8. wakeup(pq)
9. release pq->lock
```

`ip_rx()` 是 interrupt context（E1000 interrupt → `e1000_recv()` → `net_rx()` → `ip_rx()`），不能 sleep。

### sys_recv() — 使用者收封包

```text
1. 找綁定的 port slot
2. acquire pq->lock
3. while (head == tail): sleep(pq, &pq->lock)   // 等 ip_rx 入列
4. dequeue: pkt = packets[head % NPACKET]; head++
5. release pq->lock
6. 解析 UDP payload: buf + sizeof(eth) + sizeof(ip) + sizeof(udp)
7. payload_len = ntohs(udp->ulen) - sizeof(udp)
8. copyout(pagetable, user_buf, payload, payload_len)
9. copyout(pagetable, src_addr, &pkt.src_ip, ...)
10. copyout(pagetable, sport_addr, &pkt.src_port, ...)
11. kfree(pkt.buf)
12. return payload_len
```

---

## Code Sketches

### 1. e1000_transmit()

```c
int e1000_transmit(char *buf, int len) {
    acquire(&e1000_lock);

    uint32 tdt = regs[E1000_TDT];

    // Ring full if DD is not set (NIC hasn't finished this slot)
    if (!(tx_ring[tdt].status & E1000_TXD_STAT_DD)) {
        release(&e1000_lock);
        return -1;
    }

    // Free old buffer
    if (tx_ring[tdt].addr)
        kfree((void *)tx_ring[tdt].addr);

    // Install new descriptor
    tx_ring[tdt].addr   = (uint64)buf;
    tx_ring[tdt].length = len;
    tx_ring[tdt].cmd    = E1000_TXD_CMD_EOP | E1000_TXD_CMD_RS;

    // Advance tail — NIC will notice and transmit
    regs[E1000_TDT] = (tdt + 1) % TX_RING_SIZE;

    release(&e1000_lock);
    return 0;
}
```

### 2. e1000_recv()

```c
void e1000_recv(void) {
    while (1) {
        uint32 next = (regs[E1000_RDT] + 1) % RX_RING_SIZE;

        if (!(rx_ring[next].status & E1000_RXD_STAT_DD))
            break;   // no more packets ready

        char   *buf = (char *)rx_ring[next].addr;
        uint16  len = rx_ring[next].length;

        net_rx(buf, len);          // deliver up; net stack owns buf now

        char *newbuf = kalloc();   // give NIC a fresh buffer
        if (!newbuf) panic("e1000_recv: kalloc");
        rx_ring[next].addr   = (uint64)newbuf;
        rx_ring[next].status = 0;  // clear DD — hand back to NIC

        regs[E1000_RDT] = next;
    }
}
```

### 3. ip_rx() — enqueue to portqueue

```c
void ip_rx(char *buf, int len) {
    struct eth *eth = (struct eth *)buf;
    struct ip  *ip  = (struct ip  *)(eth + 1);

    if (ip->ip_p != IPPROTO_UDP) { kfree(buf); return; }

    struct udp *udp = (struct udp *)(ip + 1);
    uint16 dport  = ntohs(udp->dport);
    uint16 sport  = ntohs(udp->sport);
    uint32 src_ip = ntohl(ip->ip_src);

    // Find bound port
    acquire(&netlock);
    int slot = -1;
    for (int i = 0; i < NPORT; i++) {
        if (ports[i].bound && ports[i].port == dport) { slot = i; break; }
    }
    release(&netlock);

    if (slot == -1) { kfree(buf); return; }  // no listener, drop

    struct portqueue *pq = &ports[slot];
    acquire(&pq->lock);

    if (pq->tail - pq->head >= NPACKET) {
        release(&pq->lock);
        kfree(buf);   // queue full, drop
        return;
    }

    struct packet *pkt = &pq->packets[pq->tail % NPACKET];
    pkt->buf      = buf;
    pkt->len      = len;
    pkt->src_ip   = src_ip;
    pkt->src_port = sport;
    pq->tail++;

    wakeup(pq);
    release(&pq->lock);
}
```

### 4. sys_recv() — user dequeues

```c
uint64 sys_recv(void) {
    int dport, maxlen;
    uint64 src_addr, sport_addr, buf_addr;

    argint(0, &dport);
    argaddr(1, &src_addr);
    argaddr(2, &sport_addr);
    argaddr(3, &buf_addr);
    argint(4, &maxlen);

    // Find port slot
    acquire(&netlock);
    int slot = -1;
    for (int i = 0; i < NPORT; i++)
        if (ports[i].bound && ports[i].port == dport) { slot = i; break; }
    release(&netlock);
    if (slot == -1) return -1;

    struct portqueue *pq = &ports[slot];
    acquire(&pq->lock);
    while (pq->head == pq->tail)
        sleep(pq, &pq->lock);         // wait for ip_rx

    struct packet pkt = pq->packets[pq->head % NPACKET];
    pq->head++;
    release(&pq->lock);

    // Parse payload
    struct eth *eth  = (struct eth *)pkt.buf;
    struct ip  *ip   = (struct ip  *)(eth + 1);
    struct udp *udp  = (struct udp *)(ip  + 1);
    char *payload    = (char *)(udp + 1);
    int   plen       = ntohs(udp->ulen) - sizeof(struct udp);
    if (plen > maxlen) plen = maxlen;

    struct proc *p = myproc();
    copyout(p->pagetable, buf_addr,   payload,          plen);
    copyout(p->pagetable, src_addr,   (char *)&pkt.src_ip,   sizeof(uint32));
    copyout(p->pagetable, sport_addr, (char *)&pkt.src_port, sizeof(uint16));

    kfree(pkt.buf);
    return plen;
}
```

---

## 關鍵設計決策

### 1. Descriptor Ring Ownership Protocol

TX / RX ring 的 所有權 規則不同，這裡反過來寫是常見 bug。

**TX ring：**
- `DD = 1` 時 slot 屬於 CPU（NIC 已處理完）
- `DD = 0` 時 slot 屬於 NIC（NIC 還在用或尚未處理）
- CPU 透過寫 `TDT` 把新工作交給 NIC

**RX ring：**
- NIC 收到封包後填 slot，並把 `DD` 設成 1
- CPU 看到 `DD = 1` 後取得該 slot
- CPU 補新 buffer 後寫 `RDT`，把 slot 還給 NIC

| Register | 意義 | 寫入方 |
|----------|------|--------|
| `TDT` | TX tail — CPU 下一個要使用的 slot | CPU |
| `RDT` | RX tail — CPU 最後還給 NIC 的 slot | CPU |

### 2. 兩層鎖（netlock + pq->lock）

`netlock` 只在搜尋/修改 `ports[]` 時短暫持有，睡眠或 I/O 前會釋放。

`pq->lock` 是 per-port 鎖，配合 `sleep()`/`wakeup()` 使用。檢查 queue 與呼叫 `sleep()` 時都要持有，否則可能在 empty-check 和 `sleep()` 之間錯過 `wakeup()`。

```text
ip_rx():                      sys_recv():
  acquire(pq->lock)             acquire(pq->lock)
  enqueue packet                while empty: sleep(pq, &pq->lock)
  wakeup(pq)                    dequeue
  release(pq->lock)             release(pq->lock)
```

關鍵是兩邊使用同一個 wait channel（`pq`）做 `sleep`/`wakeup`。

### 3. Buffer Ownership Transfer

封包 buffer 會跨多個 所有權 邊界：

```text
kalloc() in e1000_init     → NIC owns (in RX ring)
e1000_recv(): net_rx(buf)  → net stack owns
ip_rx(): enqueue           → portqueue owns
sys_recv(): dequeue        → sys_recv owns
kfree() in sys_recv        → freed
```

每一段都是 所有權轉移；交出去後前一段就不該再碰該 buffer。`e1000_recv()` 要先補新 buffer 再推進 `RDT`，`sys_recv()` 則是 `copyout` 後才 `kfree`。

### 4. Byte Order：Network vs Host

network header 是 **big-endian**，x86/RISC-V host 是 **little-endian**。漏做轉換通常不會直接 crash，但會得到錯的 port/length。

```c
// Correct: always convert when reading from packet headers
uint16 dport = ntohs(udp->dport);    // network → host
uint32 src   = ntohl(ip->ip_src);
int payload_len = ntohs(udp->ulen) - sizeof(struct udp);
```

規則很簡單：寫 header 用 `htons`/`htonl`，解析 header 用 `ntohs`/`ntohl`。

### 5. ip_rx() 要丟棄，不能阻塞

`ip_rx()` 是 interrupt context，不能 sleep。若 queue 滿或沒人綁 port，直接 drop（`kfree(buf)`）並 return。

---

## 常見陷阱

### 1. 還給 NIC 前沒清 DD

消化完 RX descriptor 後，要先清 DD 再推進 `RDT`。否則 NIC 可能拿到看起來「已完成」的 descriptor。

```c
rx_ring[next].status = 0;   // clear DD
regs[E1000_RDT] = next;     // only then advance
```

### 2. 沒把 RX ring 一次清乾淨

若 `e1000_recv()` 每次 interrupt 只處理一包，RX descriptors 會堆積，後續封包會開始掉。

要 loop 到 DD 清掉為止。

### 3. Missed Wakeup：sleep 前先放 pq->lock

xv6 的 `sleep()` 必須在持有對應 lock 時呼叫。`sleep` 會以原子方式在睡眠前釋放鎖、醒來後再拿回。

**錯誤：**
```c
release(&pq->lock);
if (pq->head == pq->tail)
    sleep(pq, &pq->lock);   // ip_rx may have run between release and here!
```

**正確：**
```c
acquire(&pq->lock);
while (pq->head == pq->tail)
    sleep(pq, &pq->lock);   // sleep releases lock atomically
```

### 4. copyout 前先 free buffer

`kfree(pkt.buf)` 必須在所有 `copyout()` 之後。太早 free 會讀到已釋放記憶體。

### 5. 解析封包欄位忘記 ntohs()

UDP/IP header 裡的 port 與 length 是 network byte order。little-endian 主機直接拿來算會出現怪值。

```c
// Wrong: udp->ulen is big-endian, sizeof(struct udp) is host order
int payload_len = udp->ulen - sizeof(struct udp);

// Correct
int payload_len = ntohs(udp->ulen) - sizeof(struct udp);
```

---

## End-to-End Packet Path

```text
Sending (xv6 → external):
  net_tx_udp(buf, len, dst_ip, dport, sport)
    → fills UDP, IP, Ethernet headers
    → e1000_transmit(buf, total_len)
        → writes TX descriptor
        → advances TDT
        → NIC sends packet, sets DD

Receiving (external → xv6 行程):
  NIC receives packet → sets DD in RX ring → triggers interrupt
    → e1000_recv()
        → reads rx_ring[next], delivers via net_rx(buf, len)
        → replaces buffer, clears DD, advances RDT
    → net_rx() → ip_rx()
        → parse Ethernet+IP, check UDP
        → match dport to bound portqueue
        → enqueue packet, wakeup()
    → sys_recv() wakes up
        → dequeue, parse payload
        → copyout to user space
        → kfree(buf)
        → return payload_len
```
