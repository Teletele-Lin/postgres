# CLOG / SLRU / subtrans / multixact — 事务状态存储

> 源码：`src/backend/access/transam/{clog,slru,subtrans,multixact}.c`
> 头文件：`src/include/access/{transam,clog}.h`

---

## 1. 职责

记录"每个事务的最终命运"以及若干派生映射，供 MVCC 可见性判断查询：
- **CLOG**（commit log）：每个 XID 是已提交、已回滚还是进行中。
- **subtrans**：子事务 XID → 父事务 XID。
- **multixact**：多个事务同时（共享）锁住一行时，用一个 MultiXactId 代表这组事务。
- 这三者都建立在通用的 **SLRU**（Simple LRU）缓存框架之上。

---

## 2. XID 体系

`src/include/access/transam.h`：

```c
typedef uint32 TransactionId;          // 32 位，会回卷（wraparound）
#define InvalidTransactionId      0     // 无效                       transam.h:31
#define BootstrapTransactionId    1     // bootstrap 操作（initdb）    transam.h:32
#define FrozenTransactionId       2     // 极老元组冻结后的 xmin       transam.h:33
#define FirstNormalTransactionId  3     // 第一个"正常"XID             transam.h:34

typedef struct FullTransactionId { uint64 value; } FullTransactionId;  // epoch(32) + xid(32)
```

### 2.1 XID 回卷问题

XID 是 32 位、单调递增、用尽即回卷。可见性判断靠"XID 比较"（`TransactionIdPrecedes`），用模 2^32 的循环比较：任意 XID 的"过去一半"可见、"未来一半"不可见。但若某元组的 `t_xmin` 老到落进"未来一半"，就会突然变得不可见 → 数据消失。

**冻结（freezing）** 解决它：VACUUM 把足够老（早于某 horizon）且已提交的元组的 `t_xmin` 改写为 `FrozenTransactionId`（2），该值在循环比较中**永远是过去** → 永远可见。`relfrozenxid`/`datfrozenxid` 跟踪冻结进度；接近回卷阈值时强制 anti-wraparound autovacuum（`varsup.c`）。`FullTransactionId`（64 位 = epoch + xid）用于不能容忍回卷歧义的场景（如 WAL、快照计算）。

---

## 3. SLRU —— 通用简单 LRU 缓存

`slru.c`。CLOG/subtrans/multixact 等都是"按 XID 索引的、稠密的、远大于内存的小记录数组"，落在 `pg_xact/`、`pg_subtrans/`、`pg_multixact/` 等目录的**段文件**里。SLRU 是它们共享的缓存层：

- 把磁盘文件切成固定大小的页，在共享内存里缓存少量页（每种 SLRU 一组 buffer）。
- 简单的 LRU 替换 + per-page 状态 + LWLock（`SimpleLruReadPage`/`WritePage`）。
- 比 buffer manager 简单得多（无 pin、无哈希表、页数少），因为访问模式高度局部（新事务集中在最新页）。
- 旧段文件随 VACUUM 推进 horizon 被 `SimpleLruTruncate` 删除。

PostgreSQL 近版本把 SLRU buffer 数量做成可配置 GUC（如 `subtransaction_buffers`），缓解高并发下的 SLRU 争用。

---

## 4. CLOG —— 提交日志

`clog.c`、`src/include/access/clog.h`。每个 XID 占 **2 bit**（`clog.h:27-30`）：

```c
#define TRANSACTION_STATUS_IN_PROGRESS   0x00   // 进行中（或崩溃丢失）
#define TRANSACTION_STATUS_COMMITTED     0x01   // 已提交
#define TRANSACTION_STATUS_ABORTED       0x02   // 已回滚
#define TRANSACTION_STATUS_SUB_COMMITTED 0x03   // 子事务已提交（待父事务定论）
```

2 bit × 数十亿 XID → 几百 MB，存 `pg_xact/` 段文件，经 SLRU 缓存。

### 4.1 提交的原子时刻

事务提交时 `TransactionIdCommitTree()` 把 XID（及其已提交子事务）在 CLOG 标 `COMMITTED`。这一位的翻转就是"提交对全世界生效"的瞬间（在 commit WAL 已落盘之后，见 [30-xact](30-xact.md)）。回滚通常**不写** CLOG——崩溃后未标记的 XID 被恢复逻辑视为 aborted。

### 4.2 HINT bits：避免反复查 CLOG

可见性判断频繁，若每次都查 CLOG（哪怕有 SLRU 缓存）也贵。于是首次判定某 XID 状态后，把结果**回写到元组的 `t_infomask`** HINT bits（`HEAP_XMIN_COMMITTED`/`HEAP_XMIN_INVALID`/`HEAP_XMAX_COMMITTED`…，见 [../storage/15-page-layout](../storage/15-page-layout.md)）。之后访问该元组直接读位，无需再查 CLOG。HINT bits 是**纯缓存**：丢了可重算，所以设置 HINT bits 的脏页可以不写 WAL（除非校验和开启）。

---

## 5. subtrans —— 子事务父子映射

`subtrans.c`。子事务 XID → 直接父 XID 的映射，存 `pg_subtrans/`（每 XID 4 字节，SLRU 缓存）。用途：可见性判断遇到一个 XID 时，要沿父链找到顶层事务才能查它在 CLOG 的最终状态（子事务 `SUB_COMMITTED` 要看父事务是否真提交）。`SubTransGetTopmostTransaction()` 沿链上溯。只在有子事务时才用到。

---

## 6. multixact —— 共享行锁的多事务集合

`multixact.c`。当多个事务同时对**同一行加共享锁**（如多个 `SELECT FOR SHARE`，或外键检查），一行的 `t_xmax` 放不下多个 XID，于是分配一个 **MultiXactId** 代表"这组事务 + 各自的锁模式"，把 `t_xmax` 设为该 MultiXactId（并打 `HEAP_XMAX_IS_MULTI` 标志）。

存两组 SLRU 文件（`pg_multixact/`）：
- **offsets**：MultiXactId → 在 members 文件中的起始偏移。
- **members**：每个成员的 (XID, 锁模式)。

MultiXactId 也是 32 位、会回卷，同样需要 VACUUM 冻结（`relminmxid`）。判定一行是否被锁/可更新时，要展开 MultiXact 查每个成员的状态。multixact 是 PostgreSQL 行锁语义中最复杂的部分之一。

---

## 7. 设计模式

- **SLRU：为稠密小记录定制的轻量缓存**：放弃 buffer manager 的复杂度（pin、哈希、大池），针对"按 XID 顺序、局部性强"的访问用极简 LRU——按访问模式裁剪机制。
- **2-bit 状态 + 原子翻转**：用最小存储记录事务命运，提交即翻一位，作为全局可见性的真相源。
- **HINT bits 旁路缓存**：把昂贵的 CLOG 查询结果就地缓存进元组，可丢可重算，免 WAL——典型的"惰性物化 + 可重建缓存"。
- **回卷 + 冻结**：用循环比较省下 XID 位宽，用冻结（改写为永久过去值）+ FullTransactionId 化解回卷歧义。
- **间接代表集合（MultiXactId）**：单个 `t_xmax` 容纳多锁者的问题，用"分配一个 ID 代表一组"解决，与 [../concurrency/22](../concurrency/22-heavyweight-lock.md) 的思路一致。

---

## 8. 架构编排

```
可见性判断（HeapTupleSatisfiesMVCC，见 32）
  │ 需要 XID 的最终状态
  ├─ 先看元组 HINT bits（t_infomask）—— 命中则免查
  └─ 未命中 → TransactionIdGetStatus(xid)
        └─ 子事务？ SubTransGetTopmostTransaction（subtrans SLRU）找顶层
        └─ 查 CLOG（clog SLRU，pg_xact/）→ COMMITTED/ABORTED
        └─ 回写 HINT bits

行锁：多共享锁者 → MultiXactIdCreate → t_xmax = MXID（multixact SLRU）

提交：CommitTransaction → TransactionIdCommitTree → CLOG 标 COMMITTED
VACUUM：冻结老 xmin（FrozenTransactionId）、推进 relfrozenxid/relminmxid、Truncate SLRU 旧段
```

---

## 9. 动手探索

```sql
SELECT txid_current();                       -- 当前 XID（64 位 epoch+xid 形式）
SELECT relname, relfrozenxid, relminmxid FROM pg_class WHERE relname='t';
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;  -- 距回卷的"年龄"

-- HINT bits：首次 SELECT 后元组 infomask 出现 XMIN_COMMITTED
CREATE EXTENSION pageinspect;
SELECT t_infomask::bit(16) FROM heap_page_items(get_raw_page('t',0)) LIMIT 3;

-- SLRU 缓存统计（PG17+）
SELECT * FROM pg_stat_slru;
```

```bash
ls $PGDATA/pg_xact $PGDATA/pg_subtrans $PGDATA/pg_multixact   # SLRU 段文件
```

---

## 相关模块

- 谁来标记：[30-xact](30-xact.md)
- 谁来查询：[32-mvcc-snapshot](32-mvcc-snapshot.md)
- HINT bits 所在：[../storage/15-page-layout](../storage/15-page-layout.md)
- 冻结由谁做：[../storage/11-heap](../storage/11-heap.md)（VACUUM）
