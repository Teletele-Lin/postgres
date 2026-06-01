# 锁体系总览 — 四层并发控制

> 源码：`src/backend/storage/lmgr/`、`src/include/storage/{s_lock,lwlock,lock,proc}.h`

PostgreSQL 的并发正确性由 **MVCC + 四层锁** 共同保证。MVCC（见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)）让读写不互斥，锁体系则保护共享内存结构、串行化必要的操作、检测死锁。理解"哪层锁解决什么问题"是阅读所有并发代码的地图。

---

## 1. 四层锁的分工

| 层 | 实现 | 保护对象 | 持有时长 | 死锁检测 | 文档 |
|----|------|---------|---------|---------|------|
| **Spinlock** | `s_lock.h`（汇编 TAS） | 极短临界区（如更新一个计数器） | 纳秒~微秒 | 无 | 本文 §2 |
| **LWLock** | `lwlock.c` | 共享内存数据结构（buffer、WAL、ProcArray…） | 微秒~毫秒 | 无 | [21-lwlock](21-lwlock.md) |
| **重量级锁（Lock）** | `lock.c` | SQL 对象（表、行、事务、咨询锁） | 语句~事务 | **有** | [22-heavyweight-lock](22-heavyweight-lock.md) |
| **谓词锁（Predicate）** | `predicate.c` | SSI 的读写冲突追踪 | 事务 | SSI 算法 | [23-predicate-ssi](23-predicate-ssi.md) |

设计原则：**锁越轻量、持有越短、争用代价越低，但功能越弱（无死锁检测、无队列公平性）**。所以用最轻的能满足需求的锁——spinlock 保护 LWLock 内部状态，LWLock 保护重量级锁表，重量级锁实现 SQL 语义。

---

## 2. Spinlock —— 最底层

`src/include/storage/s_lock.h`、`spin.h`。

- **硬件 TAS（test-and-set）原语**，平台相关汇编（x86 `xchg`、ARM LL/SC 等）。
- 获取不到就**忙等（busy-wait）**，不进 OS 睡眠——因此只能保护**绝不阻塞、极短**的临界区（几条指令）。
- 无死锁检测、无公平性、不可重入。临界区内**禁止**调用任何可能出错或阻塞的代码（否则忙等的 CPU 永远转下去）。
- 典型用途：保护 LWLock 的等待队列指针、保护某些共享计数器。`SpinLockAcquire/Release`。

在现代代码里 spinlock 大量被**原子操作**取代（如 buffer 的 `state`、LWLock 的 `lockcount` 都改用 `pg_atomic_*`），spinlock 退居更窄的场景。

---

## 3. Latch —— 进程间等待/唤醒

`src/include/storage/latch.h`。严格说不是"锁"，而是**进程间信令**原语，但与锁体系紧密配合：

- `WaitLatch(latch, events, timeout)`：让进程睡眠，直到 latch 被设置、socket 可读、超时或收到信号。
- `SetLatch(latch)`：唤醒等待该 latch 的进程（可跨进程，是异步信号安全的）。
- 底层用 `epoll`/`kqueue`/`poll` + 自管道。
- 用途无处不在：LWLock 等待者被唤醒、backend 等客户端消息、checkpointer/walwriter 被唤醒干活、并行 worker 同步、复制等待。

每个 backend 在其 `PGPROC` 里有一个 latch（`procLatch`），是 backend 间"叫醒对方"的标准手段。

---

## 4. PGPROC —— 每进程的共享内存名片

`src/include/storage/proc.h`。锁体系的枢纽数据结构：每个 backend/辅助进程在共享内存中有一个 `PGPROC`，记录该进程的并发状态，供其他进程查看：

- 进程的 XID、xmin、数据库、角色。
- **正在等待的锁**（`waitLock`/`waitProcLock`）与等待模式 —— 死锁检测遍历这些字段构 waits-for 图。
- 持有的 LWLock 列表、等待的 LWLock 位置。
- `procLatch`（见 §3）。
- 子事务 XID 缓存、lock group leader（并行查询组锁）。

`ProcArray` 是所有 `PGPROC` 的数组，`GetSnapshotData` 扫它收集运行中的 XID（MVCC 快照的来源，见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)）。

---

## 5. 为什么需要这么多层？

一个具体例子串起全部四层 —— 执行 `UPDATE accounts SET bal=bal-1 WHERE id=5`：

1. **重量级锁**：对 `accounts` 表加 `RowExclusiveLock`（DML），对目标行加行锁（通过 `t_xmax` + 必要时重量级 tuple lock）。这层有死锁检测。
2. **LWLock**：读写缓冲池时加 buffer 的 content lock；查 ProcArray 取快照时加 `ProcArrayLock`；写 WAL 时加 `WALInsertLock`。
3. **Spinlock / 原子操作**：pin buffer（原子改 `state`）、更新某些共享计数器。
4. **谓词锁**（仅 SERIALIZABLE）：记录本事务读了哪些行/页，供 SSI 检测危险的读写依赖环。

每层各司其职：重量级锁管 SQL 语义与死锁，LWLock 管共享结构的短临界区，spinlock/原子管最底层状态，谓词锁管可串行化。

---

## 6. 设计模式

- **分层锁（按代价/功能权衡）**：不是一种锁打天下，而是一组从"快但弱"到"慢但强"的锁，调用方按需选最轻的——这是整个并发设计的总纲。
- **原子操作优先**：高频路径（pin buffer、共享 LWLock）尽量用 CAS/原子量替代 spinlock，消除忙等热点。
- **PGPROC 作为共享名片**：把每进程的并发状态发布到共享内存，让快照、死锁检测、唤醒都能"看到别人"。
- **Latch 解耦等待与唤醒**：用事件驱动的睡眠/唤醒替代轮询，跨进程协作的统一基元。

---

## 7. 架构编排

```
SQL 语义层：重量级锁（lock.c）——表锁/行锁/咨询锁，死锁检测（deadlock.c）
   │ 锁表本身存在共享内存，访问它需要……
   ▼
共享结构保护：LWLock（lwlock.c）——锁表分区锁、buffer content lock、WALInsertLock、ProcArrayLock
   │ LWLock 的等待队列等内部状态需要……
   ▼
最底层：Spinlock / 原子操作（s_lock.h / pg_atomic）
   ┄ 横切：Latch（睡眠/唤醒）、PGPROC（每进程名片）、ProcArray（进程数组）
可串行化层（可选）：谓词锁（predicate.c）—— SSI 读写冲突追踪
```

---

## 8. 动手探索

```sql
-- 当前持有/等待的重量级锁
SELECT locktype, relation::regclass, mode, granted, pid FROM pg_locks ORDER BY granted;

-- 正在等待什么（含 LWLock、Lock、IO 等所有等待事件）
SELECT pid, wait_event_type, wait_event, state, query FROM pg_stat_activity
WHERE wait_event IS NOT NULL;

-- 阻塞关系
SELECT blocked.pid AS blocked, blocking.pid AS blocking
FROM pg_locks blocked
JOIN pg_locks blocking ON blocked.transactionid = blocking.transactionid
WHERE NOT blocked.granted AND blocking.granted;
```

---

## 相关模块

- LWLock 细节：[21-lwlock](21-lwlock.md)
- 重量级锁与死锁：[22-heavyweight-lock](22-heavyweight-lock.md)
- SSI：[23-predicate-ssi](23-predicate-ssi.md)
- 快照来源（ProcArray）：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
