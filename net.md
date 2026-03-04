# xv6 Networking

https://pdos.csail.mit.edu/6.1810/2025/labs/net.html

## Table of Contents
1. [Background: Network Stack Layers](#background-network-stack-layers)
2. [Lab Tasks Overview](#lab-tasks-overview)
3. [Part 1: E1000 NIC Driver](#part-1-e1000-nic-driver)
4. [Part 2: UDP Receive Stack](#part-2-udp-receive-stack)
5. [Code Sketches](#code-sketches)
6. [Key Design Decisions](#key-design-decisions)
7. [Common Pitfalls](#common-pitfalls)

---

## Background: Network Stack Layers

A packet travels through layers on both send and receive:

```
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

Headers are **prepended**, so the Ethernet header is always at the start of the buffer. Parsing means casting the buffer pointer at the right offsets.

### E1000 NIC and Descriptor Rings

The E1000 is a hardware NIC (emulated by QEMU). The kernel communicates with it through **descriptor rings** in memory — circular arrays of descriptor structs that the CPU and NIC share:

```
TX ring (transmit):
  CPU writes descriptor → sets TDT → NIC reads and sends → sets DD bit

RX ring (receive):
  NIC writes descriptor + data → sets DD bit → CPU reads → sets RDT
```

`TDT` (Tail Descriptor Transmit) and `RDT` (Tail Descriptor Receive) are **NIC registers** that mark where the producer last wrote. The NIC processes from its internal head up to but not including the tail.

---

## Lab Tasks Overview

| Task | Function | Role |
|------|----------|------|
| TX driver | `e1000_transmit()` | Put packet in TX ring, notify NIC |
| RX driver | `e1000_recv()` | Drain RX ring, deliver to network stack |
| UDP bind | `sys_bind()` | Register a process to receive on a port |
| UDP recv | `sys_recv()` | Block until a packet arrives, return payload |
| IP filter | `ip_rx()` | Match incoming packets to bound ports, queue |

These five pieces form one end-to-end receive/transmit path.

---

## Part 1: E1000 NIC Driver

### Transmit: e1000_transmit()

The TX ring has `TX_RING_SIZE` (16) descriptor slots. Each slot holds:
- `addr`: physical address of the packet buffer
- `length`: packet length in bytes
- `cmd`: command flags (`EOP` = end of packet, `RS` = request status)
- `status`: set by NIC when done (`DD` = descriptor done)

The protocol:

```
1. Read TDT — this is the next slot to use
2. Check DD bit at tx_ring[TDT].status
   - If DD is clear: NIC hasn't finished previous packet in this slot → ring full, return -1
3. Free the old buffer at this slot (from a previous send)
4. Fill: addr = new buffer, length, cmd = EOP | RS
5. Advance TDT = (TDT + 1) % TX_RING_SIZE
6. The NIC will transmit and set DD when done
```

The network stack passes a `kalloc()` buffer in. After a successful enqueue, ownership moves to the driver, which later `kfree()`s it when that slot is reused.

### Receive: e1000_recv()

The RX ring works differently — `RDT` points to the **last** descriptor given to the NIC, not the next one to process. The next slot to check is `(RDT + 1) % RX_RING_SIZE`.

```
while (1):
    next = (RDT + 1) % RX_RING_SIZE
    if DD bit at rx_ring[next] is clear: break   // no more packets

    buf = rx_ring[next].addr                     // packet buffer
    len = rx_ring[next].length
    net_rx(buf, len)                              // deliver up the stack

    rx_ring[next].addr = kalloc()                // give NIC a fresh buffer
    rx_ring[next].status = 0                     // clear DD
    RDT = next                                   // tell NIC this slot is ready
```

`e1000_recv()` runs in interrupt context, so it should drain all currently ready RX descriptors in one pass.

---

## Part 2: UDP Receive Stack

### Data Structures

```c
#define NPORT   16    // max simultaneously bound ports
#define NPACKET 16    // max queued packets per port

struct packet {
    char   *buf;       // kalloc'd buffer (full Ethernet frame)
    int     len;
    uint32  src_ip;
    uint16  src_port;
};

struct portqueue {
    struct spinlock lock;
    int    bound;
    uint16 port;
    struct packet packets[NPACKET];  // circular buffer
    uint   head;    // next to dequeue
    uint   tail;    // next to enqueue
};

static struct portqueue ports[NPORT];
static struct spinlock  netlock;     // protects the ports[] array itself
```

Two lock levels are used:
- `netlock`: coarse lock for searching/modifying the `ports[]` array
- `pq->lock`: per-port lock for the packet queue (allows concurrent recv on different ports)

### sys_bind()

Associates a port number with a free `portqueue` slot:

```
1. Validate port (1–65535)
2. Acquire netlock
3. Scan ports[]: if port already bound, return 0 (idempotent)
4. Find first unbound slot
5. Set slot.bound=1, slot.port=port, reset head/tail
6. Release netlock
```

### ip_rx() — Incoming Packet Dispatcher

Called by `net_rx()` after basic Ethernet/IP parsing:

```
1. Check ip->ip_p == IPPROTO_UDP; if not, kfree(buf) and return
2. Parse UDP header: dport = ntohs(udp->dport)
3. Find matching bound port in ports[]
4. If not found: kfree(buf), return (drop)
5. Acquire pq->lock
6. If queue full (tail - head >= NPACKET): kfree(buf), return (drop)
7. Enqueue: packets[tail % NPACKET] = {buf, len, src_ip, src_port}; tail++
8. wakeup(pq)
9. Release pq->lock
```

`ip_rx()` runs in interrupt context (via the E1000 interrupt → `e1000_recv()` → `net_rx()` → `ip_rx()`). It must not sleep.

### sys_recv() — User Receives a Packet

```
1. Find bound port slot
2. Acquire pq->lock
3. while (head == tail): sleep(pq, &pq->lock)   // wait for ip_rx to enqueue
4. Dequeue: pkt = packets[head % NPACKET]; head++
5. Release pq->lock
6. Parse headers to find UDP payload: buf + sizeof(eth) + sizeof(ip) + sizeof(udp)
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

## Key Design Decisions

### 1. Descriptor Ring Ownership Protocol

The TX and RX rings use different ownership conventions — getting these backwards is a common bug.

**TX ring:**
- CPU owns a slot when `DD = 1` (NIC has finished with it)
- NIC owns a slot when `DD = 0` (NIC is still working or hasn't started)
- CPU writes to `TDT` to give new work to the NIC

**RX ring:**
- NIC fills a slot and sets `DD = 1` when a packet arrives
- CPU owns a slot after seeing `DD = 1`
- CPU writes to `RDT` to return the slot to the NIC with a fresh buffer

| Register | Meaning | Writer |
|----------|---------|--------|
| `TDT` | TX tail — next slot CPU will use | CPU |
| `RDT` | RX tail — last slot CPU returned to NIC | CPU |

### 2. Two-Level Locking (netlock + pq->lock)

`netlock` is held briefly only when searching or modifying the `ports[]` array. It is released before sleeping or doing I/O.

`pq->lock` is the per-port lock used with `sleep()`/`wakeup()`. It must be held when checking the queue and when calling `sleep()` — otherwise, a packet could arrive and call `wakeup()` between the empty-check and the `sleep()` call (missed wakeup).

```
ip_rx():                      sys_recv():
  acquire(pq->lock)             acquire(pq->lock)
  enqueue packet                while empty: sleep(pq, &pq->lock)
  wakeup(pq)                    dequeue
  release(pq->lock)             release(pq->lock)
```

The key is that both sides use the same wait channel (`pq`) for `sleep`/`wakeup`.

### 3. Buffer Ownership Transfer

Packet buffers cross multiple ownership boundaries:

```
kalloc() in e1000_init     → NIC owns (in RX ring)
e1000_recv(): net_rx(buf)  → net stack owns
ip_rx(): enqueue           → portqueue owns
sys_recv(): dequeue        → sys_recv owns
kfree() in sys_recv        → freed
```

Each step is an ownership transfer: once passed on, previous code must stop touching that buffer. `e1000_recv()` installs a replacement before advancing `RDT`, and `sys_recv()` frees only after `copyout`.

### 4. Byte Order: Network vs Host

Network headers are **big-endian**, while x86/RISC-V hosts are **little-endian**. Missing conversions usually gives silent logic bugs (wrong ports/lengths).

```c
// Correct: always convert when reading from packet headers
uint16 dport = ntohs(udp->dport);    // network → host
uint32 src   = ntohl(ip->ip_src);
int payload_len = ntohs(udp->ulen) - sizeof(struct udp);
```

Rule: use `htons`/`htonl` when writing headers, and `ntohs`/`ntohl` when parsing.

### 5. Drop, Don't Block in ip_rx()

`ip_rx()` runs from interrupt context and must not sleep. If queue is full or no port is bound, drop (`kfree(buf)`) and return.

---

## Common Pitfalls

### 1. Not Clearing DD Before Returning Slot to NIC

After consuming an RX descriptor, the DD bit must be cleared before advancing `RDT`. Otherwise the NIC hands back a descriptor that looks "done" before it has written a new packet.

```c
rx_ring[next].status = 0;   // clear DD
regs[E1000_RDT] = next;     // only then advance
```

### 2. Not Draining the Entire RX Ring

If `e1000_recv()` handles only one packet per interrupt, RX descriptors back up and later packets start dropping.

Always process in a loop until DD is clear.

### 3. Missed Wakeup: Releasing pq->lock Before sleep()

`sleep()` in xv6 must be called **while holding** the associated lock. The lock is atomically released and re-acquired around the actual sleep.

**Wrong:**
```c
release(&pq->lock);
if (pq->head == pq->tail)
    sleep(pq, &pq->lock);   // ip_rx may have run between release and here!
```

**Correct:**
```c
acquire(&pq->lock);
while (pq->head == pq->tail)
    sleep(pq, &pq->lock);   // sleep releases lock atomically
```

### 4. Freeing the Buffer Before copyout()

`kfree(pkt.buf)` has to be after all `copyout()` operations. Freeing earlier can read freed memory.

### 5. Forgetting ntohs() on Packet Fields

Port numbers and lengths in UDP/IP headers are in network byte order. Using them raw on a little-endian machine gives nonsensical values.

```c
// Wrong: udp->ulen is big-endian, sizeof(struct udp) is host order
int payload_len = udp->ulen - sizeof(struct udp);

// Correct
int payload_len = ntohs(udp->ulen) - sizeof(struct udp);
```

---

## End-to-End Packet Path

```
Sending (xv6 → external):
  net_tx_udp(buf, len, dst_ip, dport, sport)
    → fills UDP, IP, Ethernet headers
    → e1000_transmit(buf, total_len)
        → writes TX descriptor
        → advances TDT
        → NIC sends packet, sets DD

Receiving (external → xv6 process):
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
