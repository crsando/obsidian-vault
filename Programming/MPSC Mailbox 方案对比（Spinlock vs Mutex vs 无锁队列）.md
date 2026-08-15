## 背景

多生产者单消费者（**MPSC**）mailbox：多个 producer 线程往队列里塞消息，唯一一个 consumer 线程负责取出并处理。常见于订单事件分发、actor 模型的消息投递、日志聚合等场景。实现方式主要有三种：**spinlock 保护的队列**、**mutex 保护的队列**、**无锁（lock-free）intrusive 队列**（以 Dmitry Vyukov 在 1024cores.net 上发布的经典设计为代表）。

三者的根本区别在于：**producer 抢不到临界区时怎么办**，以及**是否存在"锁"这个概念本身**。

---

## 方案一：Spinlock

### 原理

抢不到锁就在一个循环里反复用 CAS/原子操作试探，直到抢到为止。全程占用 CPU，不会让出。

```c
pthread_spinlock_t lock;
pthread_spin_lock(&lock);
// 临界区：入队
pthread_spin_unlock(&lock);
```

### 优点

- 无系统调用、无上下文切换，uncontended 情况下延迟极低
- 临界区极短时（比如就是"挂一个节点到链表头"）性能非常好
- 实现简单，语义直观

### 缺点

- **lock holder preemption 问题**：持锁线程被 OS 抢占后，其他线程只能干等着空转烧 CPU，反而拖慢整机
- producer 线程数一旦超过 CPU 核数，spin 等待会严重浪费 CPU
- 在虚拟机/容器里，vCPU 可能被 hypervisor 调度走，这时候 spin 等于在等一个可能根本没在跑的 CPU，代价很大
- 没有公平性保证，容易出现某些线程被饿死

### 适用条件

- 临界区极短（几十个 cycle 级别）
- producer 线程数 ≤ CPU 核数
- 不在虚拟化/容器化环境里跑，或者能绑核

---

## 方案二：Mutex

### 原理

抢不到锁就直接睡眠（内核态，Linux 下走 futex），把 CPU 让给别的线程；锁释放后被唤醒。有上下文切换开销。

```c
uv_mutex_t lock;
uv_mutex_lock(&lock);
// 临界区：入队
uv_mutex_unlock(&lock);
```

### 优点

- 不会浪费 CPU 空转，适合临界区可能变长、或不确定并发规模的场景
- 相对公平，不容易出现某个线程一直抢不到
- 跨平台语义统一（尽管底层实现不同）

### 缺点

- uncontended 情况下也有一定固定开销（futex 系统调用，量级在几十到上百纳秒）
- contended 情况下唤醒延迟不稳定，可能到微秒级
- 对延迟敏感的高频路径（比如下单）不够激进

### 平台细节要注意

- Linux：`pthread_mutex` → futex
- Windows：`CRITICAL_SECTION` 其实是**混合方案**——先 spin 几百次，不行再睡（`InitializeCriticalSectionAndSpinCount` 可调 spin 次数），某种意义上是 spinlock 和 mutex 的折中
- libuv 的 `uv_mutex_t` 在不同平台上包了不同的底层实现，跨平台写代码时行为不完全一致

### 适用条件

- 临界区长度不确定，或可能变长
- producer 线程数可能超过核数
- 在虚拟机/容器环境，或者需要更好的公平性

---

## 方案三：无锁 intrusive MPSC 队列（Vyukov 设计）

### 原理

完全不用锁，只在 push 的时候有**一次原子操作**（`XCHG`，不是 CAS 循环）。链表是"反向"建出来的：新节点先通过 XCHG 把自己变成新的 head，再回头把旧 head 的 `next` 指向自己。

```c
struct node_t { node_t* next; /* + payload */ };
struct mpscq_t { node_t* head; node_t* tail; node_t stub; };

void push(mpscq_t* q, node_t* n) {
    n->next = NULL;
    node_t* prev = atomic_exchange(&q->head, n); // 唯一同步点，一步到位
    prev->next = n;                                // 普通 store
}

node_t* pop(mpscq_t* q) {
    node_t* tail = q->tail;
    node_t* next = tail->next;
    if (tail == &q->stub) {
        if (!next) return NULL;
        q->tail = next; tail = next; next = next->next;
    }
    if (next) { q->tail = next; return tail; }
    if (tail != q->head) return NULL;   // 撞上"缝隙"，判定暂时为空，重试
    push(q, &q->stub);
    next = tail->next;
    if (next) { q->tail = next; return tail; }
    return NULL;
}
```

### 关键机制："缝隙"（inconsistent window）

push 分两步：① `XCHG(&head, n)` ② `prev->next = n`。两步之间存在一个极短的窗口——从 head 链能看到新节点"已经入队"，但物理上 `prev->next` 这根指针还没写完。consumer 如果恰好在这个窗口里走到这个位置，会通过比较 `tail != head` 识别出"有 producer 正在写入、还没接完"，直接返回"暂时为空，稍后重试"——**不会死等，不会丢数据，也不会读到脏数据**，只是偶尔白跑一次 pop()。

stub（哨兵）节点的作用：避免"队列从空到有第一个元素"这种边界情况的特殊分支，让 push/pop 代码路径统一。

### 优点

- producer 之间几乎零竞争损耗——XCHG 保证一次成功，不像 CAS 那样撞车要重试
- 没有"持锁-释放"语义，天然没有 convoy 效应，没有 lock holder preemption 问题
- consumer 侧几乎是纯顺序代码（只有一次 load 涉及跨线程同步），cache 友好，延迟极其稳定
- 三种方案里通常是延迟最低、抖动最小的

### 缺点

- 必须是 intrusive 设计（node 自带 next 指针），内存管理要自己搞（对象池/预分配），不能随手 malloc/free，否则要处理内存复用和 ABA 之类的问题
- 严格来说不是每个瞬间都"线性一致"的——存在那个短暂的"假空"窗口，理解和写测试时要留意（不影响正确性，只是偶尔多一次 retry）
- 实现和调试难度明显高于加锁方案，出错不容易复现

### 适用条件

- 临界区就是"入队/出队"这种轻量操作，没有复杂业务逻辑塞在锁里
- 能接受 intrusive 设计和对象池化管理内存
- 对延迟和抖动有较高要求（比如交易系统的订单事件分发）

---

## 综合对比

| 维度 | Spinlock | Mutex | 无锁队列（Vyukov） |
|---|---|---|---|
| **等锁时的行为** | busy-wait 空转 | 睡眠让出 CPU | 无锁概念，仅一次原子 XCHG |
| **uncontended 延迟** | 极低 | 有固定开销（几十~上百 ns） | 最低 |
| **contended 延迟稳定性** | 差（可能被抢占） | 中等（唤醒延迟不稳定） | 最好 |
| **CPU 占用（等待时）** | 高（空转烧核） | 低（让出） | 无等待概念 |
| **producer 数 > 核数时** | 容易灾难性下降 | 表现稳定 | 表现稳定 |
| **虚拟化/容器环境** | 风险高（vCPU 被调度走时 spin 代价巨大） | 安全 | 安全 |
| **公平性** | 差，可能饿死 | 较好 | 无锁概念，天然无饿死 |
| **实现复杂度** | 低 | 低 | 高 |
| **内存管理要求** | 常规 malloc/free 即可 | 常规 malloc/free 即可 | 需要 intrusive + 对象池 |
| **正确性心智负担** | 简单 | 简单 | 需理解"缝隙"这种瞬时不一致 |
| **典型延迟量级** | 亚微秒（uncontended） | 几十 ns ~ 微秒级 | 亚微秒，抖动最小 |

---

## 选型建议

1. **临界区就是"塞消息指针"这种极轻量操作** → 优先考虑无锁队列。三者里性能最好、抖动最小，代价是实现和内存管理更麻烦。
2. **想先用锁顶一顶，producer 线程数不多、且核够用、不在虚拟化环境** → 选 spinlock。
3. **不确定并发规模，或临界区可能变长，或在容器/虚拟机里跑** → 选 mutex，最稳妥。
4. 对交易系统的订单 mailbox 这种场景（临界区轻、对延迟极度敏感），如果消息节点能池化管理，无锁队列通常是最优解——它把"抢锁"这个动作本身都省掉了。

---

## 参考来源

- Dmitry Vyukov, *"Scalable MPMC/MPSC queues"*, [1024cores.net](http://www.1024cores.net/home/lock-free-algorithms/queues/intrusive-mpsc-node-based-queue)
- libuv 文档：`uv_mutex_t` / `uv_spinlock` 相关跨平台实现说明
