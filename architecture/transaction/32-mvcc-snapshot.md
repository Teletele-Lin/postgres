# MVCC — 快照与可见性判断

> 源码：`src/backend/access/heap/heapam_visibility.c`、`src/backend/storage/ipc/procarray.c`、`src/backend/utils/time/snapmgr.c`
> 头文件：`src/include/utils/snapshot.h`

---

## 1. 职责

实现多版本并发控制的核心问题：**给定一条元组和一个快照，这条元组对当前事务是否可见？** 由两半组成：
- **快照获取**：`GetSnapshotData()` 扫描 ProcArray，捕获"此刻正在运行的事务"，定义一个一致性时间点。
- **可见性判断**：`HeapTupleSatisfiesMVCC()` 结合元组的 `t_xmin`/`t_xmax` 与快照，判定可见性。

这让"读不阻塞写、写不阻塞读"成为现实（见 [00-overview](../00-overview.md) §5）。

---

## 2. 核心数据结构：SnapshotData

`src/include/utils/snapshot.h:140`：

```c
typedef struct SnapshotData {
    SnapshotType snapshot_type;  // MVCC / SELF / ANY / DIRTY / HISTORIC_MVCC / NON_VACUUMABLE
    TransactionId xmin;          // 所有 XID < xmin 的事务都已结束 → 其结果确定可见/不可见  snapshot.h:153
    TransactionId xmax;          // 所有 XID >= xmax 都是"未来"，不可见                     snapshot.h:154
    TransactionId *xip;          // 快照时刻正在运行的 XID 列表                            snapshot.h:164
    uint32        xcnt;          // xip[] 长度                                              snapshot.h:165
    TransactionId *subxip;       // 正在运行的子事务 XID
    int32         subxcnt;
    CommandId     curcid;        // 本事务内 CID < curcid 的命令所做修改可见                snapshot.h:183
    bool          takenDuringRecovery;
    ...
} SnapshotData;
```

直觉模型——把所有 XID 分三段：

```
   已结束（< xmin）        │   进行中（xmin..xmax）      │   未来（>= xmax）
   已提交→可见             │   要查 xip[] 判断个体        │   一律不可见
   已回滚→不可见           │   在 xip 中 = 运行中 = 不可见 │
                          │   不在 xip = 已结束          │
```

`xmin`/`xmax` 是快速边界判断，`xip[]` 处理边界内"进行中"的不确定区。

### 快照类型（`snapshot.h:31`）

| 类型 | 用途 |
|------|------|
| `SNAPSHOT_MVCC` | 普通快照：只看已提交且在快照前的版本 |
| `SNAPSHOT_SELF` | 看自己事务已做的修改 |
| `SNAPSHOT_DIRTY` | 看未提交数据（唯一约束检查、`heap_lock_tuple`） |
| `SNAPSHOT_ANY` | 看所有版本（VACUUM 扫描、某些内部用） |
| `SNAPSHOT_HISTORIC_MVCC` | 逻辑解码：重建历史时刻的快照读系统表 |
| `SNAPSHOT_NON_VACUUMABLE` | VACUUM 判断元组能否回收（用 xmin 作 horizon） |

---

## 3. 快照获取：GetSnapshotData

`GetSnapshotData()`（`procarray.c:2114`）：加 `ProcArrayLock`（共享），遍历 **ProcArray**（所有 PGPROC，见 [../concurrency/20-lock-overview](../concurrency/20-lock-overview.md)），把每个正在运行事务的 XID 收进 `xip[]`，计算 `xmin`（最小运行 XID）和 `xmax`（下一个待分配 XID）。

这是热点路径（每个语句/事务都要取快照），高并发下 ProcArrayLock 曾是瓶颈。优化：
- **快照复用**（`GetSnapshotDataReuse`，`procarray.c:2034`）：若自上次取快照以来无事务提交，直接复用旧快照，免遍历。
- 把每进程的 xmin 等"摘要"维护在紧凑数组里，减少遍历开销。

REPEATABLE READ/SERIALIZABLE 在事务开始取**一个**快照用到底；READ COMMITTED 每条语句取**新**快照（故能看到期间已提交的别人改动）。

---

## 4. 可见性判断：HeapTupleSatisfiesMVCC

`HeapTupleSatisfiesMVCC()`（`heapam_visibility.c:939`）是 MVCC 的心脏。逻辑骨架（结合 HINT bits 加速）：

### 4.1 先判 xmin（这个版本"生出来"了吗）

```c
if (!HeapTupleHeaderXminCommitted(tuple)) {       // HINT 未标记"xmin 已提交"
    if (HeapTupleHeaderXminInvalid(tuple)) return false;  // 已知 xmin 无效 → 不可见
    ... 若 xmin 是当前事务：按 cmin 与 snapshot->curcid 比较（自己刚插的，看命令序）...
    else if (XidInMVCCSnapshot(xmin, snapshot))   // xmin 在运行中 → 对我不可见  visibility.c:1005
        return false;
    else if (TransactionIdDidCommit(xmin))         // 查 CLOG：已提交
        SetHintBits(HEAP_XMIN_COMMITTED);          // 回写 HINT bits 加速下次   visibility.c:1008
    else { SetHintBits(HEAP_XMIN_INVALID); return false; }  // 回滚 → 永不可见
}
// 走到这里：xmin 已提交且在快照之前 → 这个版本对我"已存在"
```

要点：
- `XidInMVCCSnapshot`（`snapshot.h:33` 提及）用 `xmin/xmax/xip` 判断某 XID 在快照时刻是否运行中。
- `TransactionIdDidCommit` 查 CLOG（见 [31-clog-slru](31-clog-slru.md)），结果回写 HINT bits（`SetHintBits`），后续访问免查。
- 当前事务自己插入的元组按 `cmin` vs `snapshot->curcid` 判断（命令级可见性，`visibility.c:965`）。

### 4.2 再判 xmax（这个版本"被删/被锁"了吗）

```c
if (tuple->t_infomask & HEAP_XMAX_INVALID) return true;   // 没被删 → 可见
if (HEAP_XMAX_IS_LOCKED_ONLY(infomask)) return true;       // 只是被锁、没被删 → 可见
if (infomask & HEAP_XMAX_IS_MULTI) { ... 展开 MultiXact 取真正的删除者 xmax ... }
若 xmax 是当前事务：按 cmax vs curcid 判断
else if (XidInMVCCSnapshot(xmax)) return true;             // 删除者还在运行 → 对我未删 → 可见
else if (TransactionIdDidCommit(xmax)) { SetHintBits(HEAP_XMAX_COMMITTED); return false; } // 已删 → 不可见
else return true;                                          // 删除者回滚 → 仍可见
```

直觉：一个版本可见 ⟺ **它的创建事务在我之前提交，且它的删除事务（若有）尚未在我之前提交**。

`HeapTupleSatisfiesVisibility()`（`visibility.c:1732`）按 `snapshot_type` 分发到 MVCC/DIRTY/ANY 等具体函数；批量版 `HeapTupleSatisfiesMVCCBatch`（`visibility.c:1690`）对整页元组一次判定，减少函数调用开销。

---

## 5. 隔离级别

`src/include/access/xact.h`：

```c
#define XACT_READ_UNCOMMITTED 0   // PG 中等同 READ COMMITTED（无脏读）
#define XACT_READ_COMMITTED   1   // 每条语句一个新快照
#define XACT_REPEATABLE_READ  2   // 整个事务一个快照（= 快照隔离）
#define XACT_SERIALIZABLE     3   // 快照隔离 + SSI 谓词锁
```

- **READ COMMITTED**：语句级快照。UPDATE/DELETE 遇到并发已提交修改时走 **EvalPlanQual**：重新取被改行的最新版本、对其重跑 qual，决定是否仍修改（避免丢失更新，见 [../query/08-executor-nodes](../query/08-executor-nodes.md)）。
- **REPEATABLE READ**：事务级快照，看不到事务开始后别人的提交；写冲突报 `40001`。
- **SERIALIZABLE**：在 RR 之上加 SSI（见 [../concurrency/23-predicate-ssi](../concurrency/23-predicate-ssi.md)）。

---

## 6. 死元组回收的 horizon

VACUUM 能回收一个死元组，当且仅当**没有任何现存快照可能再看到它**。这由全局 **xmin horizon**（所有活动事务 xmin 的最小值，由 ProcArray 算出）界定：`t_xmax` 早于 horizon 且已提交的版本可回收。长事务/未关闭的 `idle in transaction` 会压低 horizon → 死元组堆积（膨胀）。这把 [30-xact](30-xact.md) 的事务生命周期、本模块的快照、[../storage/11-heap](../storage/11-heap.md) 的 VACUUM 三者联系起来。

---

## 7. 设计模式

- **版本可见性 = 两个 XID + 一个快照**：把并发可见性归约为对 `t_xmin`（创建）和 `t_xmax`（删除）相对快照的判定，简洁且与隔离级别正交。
- **三段式快照（xmin/xmax/xip）**：用两个边界做 O(1) 快速判定，仅对边界内"进行中"的少数 XID 查列表，平衡精度与速度。
- **HINT bits 旁路**：把 CLOG 查询结果就地缓存进元组，使稳态可见性判断几乎不查 CLOG（见 [31](31-clog-slru.md)）。
- **快照类型策略化**：同一套元组头服务普通读、脏读、全读、历史读、VACUUM horizon，靠 `snapshot_type` 分发。
- **EvalPlanQual 处理写写竞争**：READ COMMITTED 下"读到旧快照但要改最新行"的矛盾，用"重取最新版重判 qual"化解，而非加锁阻塞。
- **horizon 驱动回收**：用全局最小 xmin 作为"无人再看得到"的安全线，统一界定可冻结/可回收。

---

## 8. 架构编排

```
语句执行
  ├─ GetTransactionSnapshot / GetLatestSnapshot
  │     └─ GetSnapshotData（扫 ProcArray，ProcArrayLock 共享）  procarray.c:2114
  │           └─ 可复用则 GetSnapshotDataReuse                  procarray.c:2034
  ▼
扫描每条元组（heap_getnextslot）
  └─ HeapTupleSatisfiesVisibility → HeapTupleSatisfiesMVCC      visibility.c:939
        ├─ 判 xmin：XidInMVCCSnapshot / TransactionIdDidCommit(查CLOG) → SetHintBits
        └─ 判 xmax：同上（含 MultiXact 展开）
        → 可见则输出
写写竞争（RC）：TM_Updated → EvalPlanQual 重取最新版重判
VACUUM：以全局 xmin horizon 判定死元组可回收 → 冻结/回收
```

---

## 9. 动手探索

```sql
-- 同一行的多个版本（更新后旧版本仍在，靠可见性区分）
SELECT ctid, xmin, xmax, * FROM t;          -- 系统列 xmin/xmax
BEGIN; UPDATE t SET v=v+1 WHERE id=1;
-- 另一会话此时仍看到旧 v（旧版本对它可见）

-- 隔离级别对比
BEGIN ISOLATION LEVEL REPEATABLE READ;
  SELECT count(*) FROM t;   -- 之后别的会话插入并提交，这里再查仍是旧值
COMMIT;

-- horizon / 膨胀诱因
SELECT pid, state, age(backend_xmin), xact_start
FROM pg_stat_activity WHERE backend_xmin IS NOT NULL;  -- 长事务压低 horizon
SELECT n_dead_tup, n_live_tup FROM pg_stat_user_tables WHERE relname='t';
```

---

## 相关模块

- XID 状态：[31-clog-slru](31-clog-slru.md)
- 事务边界：[30-xact](30-xact.md)
- 元组头：[../storage/15-page-layout](../storage/15-page-layout.md)
- 回收：[../storage/11-heap](../storage/11-heap.md)
- 可串行化：[../concurrency/23-predicate-ssi](../concurrency/23-predicate-ssi.md)
- 快照来源 ProcArray：[../concurrency/20-lock-overview](../concurrency/20-lock-overview.md)
