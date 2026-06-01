# 重量级锁与死锁检测

> 源码：`src/backend/storage/lmgr/lock.c`、`deadlock.c`、`proc.c`
> 头文件：`src/include/storage/{lock,lockdefs,proc}.h`
> 上层总览：[20-lock-overview](20-lock-overview.md)

---

## 1. 职责

重量级锁（heavyweight lock，又称 regular lock）实现 **SQL 级别的并发语义**：表锁、行锁、事务锁、咨询锁等。它是四层锁里唯一具备：
- **丰富的锁模式**（8 种，带相容矩阵）；
- **死锁检测**（构建 waits-for 图查环）；
- **等待队列与公平性**；
- **跨事务持有**（多数表锁持有到事务结束）。

代价是重——加锁要查/改共享内存锁表（受 LWLock 保护），故只用于 SQL 对象级，不用于高频内部结构。

---

## 2. 锁模式与相容矩阵

`src/include/storage/lockdefs.h:36-45`，8 种模式由弱到强：

```c
#define AccessShareLock          1  // SELECT
#define RowShareLock             2  // SELECT FOR UPDATE/SHARE
#define RowExclusiveLock         3  // INSERT/UPDATE/DELETE
#define ShareUpdateExclusiveLock 4  // VACUUM(非FULL)/ANALYZE/CREATE INDEX CONCURRENTLY
#define ShareLock                5  // CREATE INDEX(非并发)
#define ShareRowExclusiveLock    6  // 类 EXCLUSIVE 但允许 ROW SHARE
#define ExclusiveLock            7  // 阻塞 ROW SHARE / SELECT FOR UPDATE
#define AccessExclusiveLock      8  // ALTER/DROP TABLE/VACUUM FULL/TRUNCATE
```

**相容矩阵**（`lock.c` 的 `LockConflicts[]`）决定两个模式能否同时持有。关键直觉：
- `AccessShareLock`（读）与 `RowExclusiveLock`（写）**相容** —— 这就是 MVCC 下"读写不互斥"在锁层面的体现：SELECT 和 INSERT/UPDATE/DELETE 同表可并发。
- `AccessExclusiveLock`（DDL）与**一切**冲突 —— `ALTER TABLE` 要独占。
- 两个 `RowExclusiveLock` 相容 —— 多个写者可并发改同一张表（行级冲突另由行锁/`t_xmax` 处理）。

注意：**表级 `RowExclusiveLock` 不锁具体行**，行级冲突由元组的 `t_xmax` + 必要时的重量级 **tuple lock**（`LOCKTAG_TUPLE`）处理。

---

## 3. 核心数据结构

### 3.1 LOCKTAG —— 锁住什么

`src/include/storage/lock.h`，用 `locktag_type` 区分锁对象种类：

| 类型 | 锁对象 | 典型场景 |
|------|--------|---------|
| `LOCKTAG_RELATION` | 整个关系 | 表锁（SELECT/DML/DDL） |
| `LOCKTAG_RELATION_EXTEND` | 关系扩展权 | 并发扩展新页时串行化 |
| `LOCKTAG_PAGE` | 某页 | 部分索引操作 |
| `LOCKTAG_TUPLE` | 某元组 | 行锁的重量级表示 |
| `LOCKTAG_TRANSACTION` | 某 XID | 等某事务结束（行锁等待常转化为此） |
| `LOCKTAG_VIRTUALTRANSACTION` | 虚拟事务 ID | |
| `LOCKTAG_OBJECT` | 任意系统对象 | DDL 对 schema/类型等加锁 |
| `LOCKTAG_ADVISORY` | 用户咨询锁 | `pg_advisory_lock()` |

### 3.2 锁表三件套（共享内存哈希表）

- **LOCK**：一个被锁对象（按 LOCKTAG 哈希），记录各模式的持有/等待总数、等待队列。
- **PROCLOCK**：(LOCK, PGPROC) 的交点 —— "某进程对某对象持有了哪些模式"。
- **LOCALLOCK**：进程**本地**的锁计数缓存，记录本进程已持有的锁及次数，使重复加同一锁（同一事务多次 SELECT 同表）无需每次都进共享锁表，是重要的快路径优化。

锁表按 `NUM_LOCK_PARTITIONS`（16）分区，每区一把 `LockManagerLWLock`（见 [21-lwlock](21-lwlock.md)）。

### 3.3 fast-path 锁

弱表锁（`AccessShareLock`/`RowShareLock`/`RowExclusiveLock`，即不冲突 DDL 的那些）走 **fast-path**：记录在 PGPROC 的本地数组里，不进共享锁表，避免分区锁争用。只有当有人请求强锁（会与 fast-path 锁冲突）时，才把 fast-path 锁"提升"到共享锁表。这让常规 DML/SELECT 的加锁几乎无共享内存争用。

---

## 4. 核心算法

### 4.1 加锁流程

`LockAcquire()`（`lock.c`）：

```
1. 查 LOCALLOCK：本进程已持有该锁？ 是 → 计数+1，立即返回（最快路径）
2. 可走 fast-path（弱锁且无强锁冲突）？ → 记进 PGPROC fast-path 数组，返回
3. 否则：加 LockManagerLWLock(对应分区)
   ├─ 查/建 LOCK 与 PROCLOCK
   ├─ 与已持有模式相容？ 是 → 授予，granted++
   └─ 冲突 → 入等待队列，设置 waitLock/waitProcLock，放 LWLock
            → 睡眠（WaitLatch），等待持有者释放时被唤醒或被死锁检测中止
```

### 4.2 死锁检测：waits-for 图

冲突等待不是立刻报死锁，而是**先睡 `deadlock_timeout`（默认 1s）**，超时仍未拿到锁才触发 `DeadLockCheck()`（`deadlock.c:19`）。这是因为多数等待会很快自然解除，死锁检测较贵，故延迟触发。

`DeadLockCheck` 的算法（`deadlock.c` 的 `FindLockCycle`，`:82`）：

1. 构建 **waits-for 图**：节点是进程（PGPROC），边 `A→B` 表示"A 等待的锁被 B 持有"（`deadlock.c:38` "One edge in the waits-for graph"）。等待信息全在各 PGPROC 的 `waitLock`/`waitProcLock` 字段里。
2. 从当前进程出发 **DFS 找环**（`FindLockCycleRecurse`）。
3. 找到环 → 当前进程作为"牺牲者"被中止：`ereport(ERROR, errcode(ERRCODE_T_R_DEADLOCK_DETECTED))`，其事务回滚释放锁，打破环。
4. 无环 → 可能只是普通等待，继续睡；若发现可通过**重排等待队列**消解（软边），则调整队列避免无谓阻塞。

死锁检测还要考虑**并行查询锁组**（lock group）：同一并行组的 leader 与 worker 视作可共享锁，检测时按组而非单进程处理（`deadlock.c` 的 group 逻辑），避免组内进程互相"假死锁"。

### 4.3 释放

事务结束时 `LockReleaseAll()` 释放本事务持有的所有重量级锁（多数表锁持有到此刻才放，保证从分析到提交表结构稳定）。`ROLLBACK`/`ERROR` 同样经此路径兜底释放（见 [../infra/71-error-elog](../infra/71-error-elog.md)）。`AccessExclusiveLock` 在备库回放时也需重建（通过 WAL 中的 `xl_standby` 记录，见 [../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md)）。

---

## 5. 设计模式

- **相容矩阵编码语义**：把"谁与谁冲突"抽成 8×8 矩阵，加锁逻辑与策略分离；MVCC 的"读写不互斥"直接体现为读锁与写锁相容。
- **三级缓存（LOCALLOCK→fast-path→共享锁表）**：绝大多数加锁命中进程本地或 fast-path，避开共享锁表与分区锁，把"常见情况做快"。
- **延迟死锁检测（先超时再查图）**：用 `deadlock_timeout` 把昂贵的环检测推迟到"真的卡住"时，乐观假设多数等待会自然解除。
- **waits-for 图 + DFS 查环**：把死锁判定规约为有向图找环，牺牲一个事务打破环——经典且可证明终止。
- **锁组（并行感知）**：把并行 worker 与 leader 视作一个加锁主体，避免内部"自死锁"。

---

## 6. 架构编排

```
SQL 执行：table_open(rel, RowExclusiveLock) / LockTuple / pg_advisory_lock
  │ LockAcquire                                         lock.c
  ├─ LOCALLOCK 命中？ → 本地计数
  ├─ fast-path（弱锁）？ → PGPROC 数组
  └─ 共享锁表（LockManagerLWLock 分区）
        ├─ 相容 → granted
        └─ 冲突 → 入队、睡眠（WaitLatch）
                    │ 超过 deadlock_timeout
                    ▼
                 DeadLockCheck → 构 waits-for 图 → FindLockCycle(DFS)
                    ├─ 有环 → 牺牲本事务（ERROR: deadlock detected）
                    └─ 无环 → 继续等 / 重排队列
释放：LockReleaseAll（提交/回滚/异常时）
```

---

## 7. 动手探索

```sql
-- 制造死锁观察（两个会话交叉加锁）
-- 会话1: BEGIN; UPDATE t SET x=1 WHERE id=1;
-- 会话2: BEGIN; UPDATE t SET x=1 WHERE id=2;
-- 会话1:        UPDATE t SET x=1 WHERE id=2;   -- 等待
-- 会话2:        UPDATE t SET x=1 WHERE id=1;   -- → deadlock detected

SHOW deadlock_timeout;

-- 当前锁与等待
SELECT locktype, relation::regclass, mode, granted, pid, fastpath FROM pg_locks;

-- 阻塞链
SELECT pg_blocking_pids(pid), pid, wait_event, query
FROM pg_stat_activity WHERE wait_event_type='Lock';
```

---

## 相关模块

- 总览：[20-lock-overview](20-lock-overview.md)
- 锁表的保护锁：[21-lwlock](21-lwlock.md)
- 行锁与 MVCC：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
- 可串行化（更强一致性）：[23-predicate-ssi](23-predicate-ssi.md)
- 异常释放：[../infra/71-error-elog](../infra/71-error-elog.md)
