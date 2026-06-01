# LWLock — 轻量级读写锁

> 源码：`src/backend/storage/lmgr/lwlock.c`、`src/include/storage/lwlock.h`
> 上层总览：[20-lock-overview](20-lock-overview.md)

---

## 1. 职责

LWLock（lightweight lock）是保护**共享内存数据结构**的读写锁。它比重量级锁轻得多（无死锁检测、无丰富的锁模式、无 SQL 可见性），但比 spinlock 强：争用时可在 OS 中睡眠而非忙等。几乎所有共享内存结构都靠 LWLock 保护：缓冲池、WAL 缓冲、ProcArray、CLOG、锁表分区、复制槽等。

只有两种模式：**共享（LW_SHARED，读）** 与 **独占（LW_EXCLUSIVE，写）**。

---

## 2. 演进：为什么不再用"spinlock 保护的 rw 锁"

`lwlock.c` 顶部注释（`lwlock.c:NOTES`）记录了关键演进：

> 早期实现是直白的读写锁，内部状态由一个 spinlock 保护。但对**高频共享加锁**的负载，spinlock 开销过高——大家在（本质独占的）spinlock 上自旋，只为获取一个其实空闲的共享锁。

于是改为**对未被独占持有的锁提供 wait-free 的共享获取**（`lwlock.c:19-20`）。这是高并发设计的经典教训：保护"读多写少"结构时，连保护元数据的那把小锁都会成为瓶颈，必须用原子操作消除它。

---

## 3. 核心数据结构

`src/include/storage/lwlock.h`：

```c
typedef struct LWLock {
    uint16          tranche;        // 所属 tranche（锁的"种类"编号，用于统计/命名）
    pg_atomic_uint32 state;         // ★lockcount：打包了 持有状态 + 标志
    proclist_head   waiters;        // 等待者队列（PGPROC 链）
} LWLock;
```

**`state` 是一个原子变量**（`lwlock.c:22-30`）：用单一 `lockcount` 取代过去分离的"共享计数"和"独占标志"。
- 独占持有：CAS 换入哨兵值 `LW_VAL_EXCLUSIVE`。
- 共享持有：计数持有者个数（每个 +1）。
- 哨兵值与最大共享持有数不冲突，靠 `MAX_BACKENDS` 限制保证（`lwlock.c:36-38`）。

每个 LWLock 被填充到 `LWLOCK_PADDED_SIZE`（对齐到缓存行 `PG_CACHE_LINE_SIZE`，通常 64B），避免**伪共享（false sharing）**——相邻锁落在同一缓存行会让无关的加锁互相拖累。

### tranche —— 锁的分类

同一用途的一组 LWLock 归为一个 **tranche**（如所有 buffer content lock 是一个 tranche）。`LWLockRegisterTranche()` 给 tranche 命名，`pg_stat_activity` / `wait_event` 据此显示"在等哪种 LWLock"（如 `WALWrite`、`BufferContent`、`LockManager`）。

### 关键分区锁组

为降低争用，热点结构的 LWLock 按 hash 分成多份：
- `BufferMappingLocks`（128 分区，见 [../storage/16-buffer-manager](../storage/16-buffer-manager.md)）。
- `WALInsertLocks`（多个，并行 WAL 插入，见 [../transaction/33-wal](../transaction/33-wal.md)）。
- `LockManagerLWLocks`（16 分区，保护重量级锁表，见 [22-heavyweight-lock](22-heavyweight-lock.md)）。
- `ProcArrayLock`、`XidGenLock`、`CLogControlLock` 等单例锁。

---

## 4. 核心算法

### 4.1 无锁共享获取

获取用对 `lockcount` 的 **CAS（compare-and-exchange）**（`lwlock.c:28-30`）：独占换入 `LW_VAL_EXCLUSIVE`，共享则原子地把持有者计数 +1。释放用**原子减**（`lwlock.c:32-34`）：减到 0 就知道该唤醒等待者。对未被独占的锁，共享获取**完全无锁、无等待**。

### 4.2 四阶段加锁（解决排队竞态）

朴素做法有个致命竞态（`lwlock.c:41-45`）：原子尝试失败 → 决定排队，但等你排进队列时，原锁持有者可能**已经释放完**了，于是你白白睡死在 OS 里没人叫醒。

解法是**两阶段确认式排队**（`lwlock.c:47-55`），实际是四步循环：

```
Phase 1: 原子尝试获取；成功 → 完事
Phase 2: 把自己加入锁的等待队列
Phase 3: 再次原子尝试获取；成功 → 把自己移出队列，完事
Phase 4: 睡眠等待唤醒；醒来 → goto Phase 1
```

关键在 **Phase 2 先入队、Phase 3 再重试**：一旦完成 Phase 2 就已在队列里，此后任何释放者都能看到并唤醒我，于是"释放得太快导致漏唤醒"不可能发生（`lwlock.c:53-55`）。这是无锁同步里"先发布意图、再检查条件"的标准手法。

### 4.3 唤醒与公平

释放时若 `lockcount` 归零且有等待者，释放者负责唤醒队首（独占等待者）或一批共享等待者，通过 `SetLatch` 叫醒对应 PGPROC（见 [20-lock-overview](20-lock-overview.md) §3）。LWLock 大体 FIFO 但允许一些重排以提高吞吐，**不保证严格公平**，也**不检测死锁**——因此使用 LWLock 的代码必须保证**固定的加锁顺序**来避免死锁（这是开发约定，不是运行时检查）。

### 4.4 错误处理集成

LWLock 持有被记录在进程本地，`ereport(ERROR)` 触发的 `LWLockReleaseAll()` 会在事务/子事务回滚时释放本进程持有的所有 LWLock（见 [../infra/71-error-elog](../infra/71-error-elog.md)），防止异常路径漏放导致系统卡死。

---

## 5. 设计模式

- **原子量取代元数据锁**：把锁状态压进一个原子 `lockcount`，用 CAS/原子减替代"保护锁的 spinlock"，对读多写少结构消除元数据争用——LWLock 演进史的核心教训。
- **先入队后重试（发布-检查）**：四阶段加锁用"先把自己放进等待队列再做最后一次尝试"消除"释放太快漏唤醒"的竞态。
- **缓存行填充防伪共享**：每锁对齐到 64B，避免相邻锁互相拖累——共享内存高并发结构的通用手法。
- **分区降热点**：对 buffer 映射、WAL 插入、锁表等热点用 N 路分区，把单点争用摊开。
- **tranche 命名促可观测**：给锁分类命名，让等待事件可被 `pg_stat_activity` 归因。
- **约定式无死锁**：放弃运行时死锁检测换取轻量，靠"全局固定加锁顺序"的开发纪律保证安全。

---

## 6. 架构编排

```
调用方（buffer/WAL/ProcArray/CLOG/锁表 …）
  │ LWLockAcquire(lock, LW_SHARED|LW_EXCLUSIVE)
  ▼  lwlock.c
Phase1 CAS(state) ── 成功 → 持有
   │ 失败
Phase2 入 waiters 队列（proclist）
Phase3 再 CAS ── 成功 → 出队、持有
   │ 失败
Phase4 WaitLatch 睡眠 ← 释放者 SetLatch 唤醒 → goto Phase1
释放：LWLockRelease → 原子减 state → 归零则唤醒队首（SetLatch）
异常：LWLockReleaseAll（回滚时兜底释放）
```

---

## 7. 动手探索

```sql
-- 看正在等待哪种 LWLock（wait_event_type='LWLock'）
SELECT pid, wait_event, query FROM pg_stat_activity
WHERE wait_event_type = 'LWLock';

-- 常见 LWLock 等待事件：WALInsert, WALWrite, BufferContent,
-- LockManager, ProcArray, CLogControl, XidGen ...
```

调试：在 `LWLockAcquire` 下条件断点（按 tranche 名）；`p lock->state` 查看 lockcount（`LW_VAL_EXCLUSIVE` 表示被独占）。`perf` 火焰图中大量 `LWLockAcquire` 自旋/睡眠通常指向某个分区不够多的热点锁。

---

## 相关模块

- 总览：[20-lock-overview](20-lock-overview.md)
- 主要使用者：[../storage/16-buffer-manager](../storage/16-buffer-manager.md)、[../transaction/33-wal](../transaction/33-wal.md)
- 重量级锁（用 LWLock 保护其锁表）：[22-heavyweight-lock](22-heavyweight-lock.md)
- 错误处理释放：[../infra/71-error-elog](../infra/71-error-elog.md)
