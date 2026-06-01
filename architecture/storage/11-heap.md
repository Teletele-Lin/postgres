# Heap — 默认表存储引擎与 HOT

> 源码：`src/backend/access/heap/`
> 设计文档：`src/backend/access/heap/README.HOT`
> 实现的接口：[10-table-am](10-table-am.md) 的 `TableAmRoutine`

---

## 1. 职责

`heap` 是 PostgreSQL 默认的 Table AM 实现：把元组以**追加式、无序**的方式存放在固定大小（默认 8KB）的页面里，配合 MVCC 多版本机制。它实现了 `TableAmRoutine` 的全部回调，并负责：元组的增删改、顺序扫描、索引回表、HOT 优化、单页 prune、Lazy VACUUM、抽样（ANALYZE）、CLUSTER 重写。

"heap" 在数据库术语里指"无序堆"，与数据结构里的堆无关——行按插入/空闲空间放置，顺序不保证。

---

## 2. 目录结构

```
src/backend/access/heap/
  heapam.c             heap_insert/update/delete/getnextslot 等核心
  heapam_handler.c     把 heap_* 包装成 TableAmRoutine 虚表（heap_tableam_handler）
  heapam_visibility.c  MVCC 可见性判断（见 transaction/32）
  hio.c                Heap Insert：RelationGetBufferForTuple 找放置页面
  pruneheap.c          单页 HOT prune / 行版本回收
  vacuumlazy.c         Lazy VACUUM（堆 + 索引 + VM/FSM 维护）
  visibilitymap.c      Visibility Map（all-visible/all-frozen 位图）
  rewriteheap.c        CLUSTER / VACUUM FULL 的全表重写
```

---

## 3. 核心数据结构

### 3.1 页面与元组

heap 页面布局、`HeapTupleHeaderData`（`t_xmin`/`t_xmax`/`t_ctid`/`t_infomask`）详见 [15-page-layout](15-page-layout.md)。这里只强调几个对 heap 逻辑至关重要的字段：

- `t_xmin` / `t_xmax`：创建/删除该版本的事务 XID —— MVCC 可见性的依据。
- `t_ctid`：通常指向自己；UPDATE 后指向**新版本**的位置，串成"更新链"。
- `t_infomask` 的 HINT bits：`HEAP_XMIN_COMMITTED`/`HEAP_XMAX_INVALID` 等，缓存事务提交状态，避免反复查 CLOG。
- `t_infomask2` 的 `HEAP_HOT_UPDATED` / `HEAP_ONLY_TUPLE`：标记 HOT 链。

### 3.2 ItemId 的间接层

每个元组通过页内的 **line pointer（ItemId）** 间接寻址，`lp_flags` 区分：`LP_NORMAL`（指向元组）、`LP_REDIRECT`（重定向到同页另一 ItemId，HOT 链用）、`LP_DEAD`（已死、空间可回收）、`LP_UNUSED`。这层间接性让元组能在页内移动而不改变其逻辑 TID，是 HOT prune 的基础。

---

## 4. 核心算法

### 4.1 插入：heap_insert

`heap_insert()`（`heapam.c:2004`）：

1. `RelationGetBufferForTuple()`（`hio.c:500`）借助 **FSM**（Free Space Map）找一个有足够空闲空间的页面（找不到则扩展关系新增页），返回已加 pin + 排他内容锁的 buffer。
2. 设置元组的 `t_xmin = 当前 XID`、`t_cid`，`PageAddItem` 放入页面。
3. `MarkBufferDirty`，写 WAL（`xl_heap_insert`），设置页 LSN。
4. **索引插入由上层执行器负责**（`execIndexing.c`），不在 `heap_insert` 内。

`multi_insert`（`heapam.c:2360` 附近）批量版本，COPY 用它一次放多行、合并 WAL，显著加速批量导入。

### 4.2 删除：heap_delete

`heap_delete()`：不物理删除，只把目标元组的 `t_xmax = 当前 XID`、设相应 infomask，写 WAL。该版本对"删除事务之后开始的快照"不可见，但旧快照仍能看到——这正是 MVCC"读不阻塞写"的代价：死元组留待 VACUUM 回收。

### 4.3 更新：heap_update 与 HOT

`heap_update()`：在旧元组上标 `t_xmax = 当前 XID`，在某页插入新版本，旧元组 `t_ctid` 指向新版本（更新链）。关键分支是**是否走 HOT**：

- **非 HOT 更新**：新版本放到（可能不同的）页面，且**所有索引都要插入指向新版本的新条目**。
- **HOT 更新**（`README.HOT`）：当 UPDATE **未修改任何被索引列**且新版本能放进**同一页**时，新版本是"Heap-Only Tuple"——**不在任何索引里建新条目**。索引仍指向旧版本（链首），通过页内 `t_ctid` / `LP_REDIRECT` 链找到最新可见版本。

### 4.4 HOT 解决的问题

`README.HOT:6-9` 点明 HOT 的双重收益：

```
HOT 消除冗余索引条目，并允许在不做全表 vacuum 的情况下
回收 DELETE/过期 UPDATE 占用的空间——靠的是单页 vacuum（即"碎片整理"或"prune"）。
```

为什么单页回收过去不可行？`README.HOT:18-32`：标准 vacuum 要清理指向死元组的索引条目，得扫描整个索引来摊薄成本；想"只回收几条"就得重算索引键去索引里找条目，而函数索引里**可能有不可靠的用户函数**——一个声称 immutable 实则不是的函数会让你找不到索引条目，造成索引损坏。所以 vacuum 宁愿**完全不调用用户代码**。

HOT 的破解之道：**更新链对索引透明**。索引只指向链首，HOT 更新不碰索引，于是回收链上的死版本时也无需碰索引——`heap_page_prune()`（`pruneheap.c`）可以纯粹在单页内整理：把死版本的 `LP_NORMAL` 改 `LP_DEAD`/`LP_UNUSED`，把链首 redirect 到第一个活版本，**无需任何索引操作、无需用户代码**。

`README.HOT:34-46` 给出 HOT 适用的两类情形：
1. 元组反复更新但**不改任何被索引列**（"被索引列"包括 partial index 谓词里引用但未存储的列）。
2. 修改的列只被"不含 TID、按块汇总"的索引（如 BRIN）使用——这类索引无单行引用，无需新建 HOT 链，只需被告知新数据。

### 4.5 顺序扫描：heap_getnextslot

`heap_getnextslot()`（`heapam.c:1474`）逐页遍历：用 `ReadStream` 预读下一批页面（见 [17-smgr-forks](17-smgr-forks.md)），对每页每个 `LP_NORMAL` 元组调可见性判断 `HeapTupleSatisfiesMVCC`（见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)），可见则存入 slot 返回。

### 4.6 Lazy VACUUM

`vacuumlazy.c` 的两/三阶段：
1. **扫堆**（按 VM 跳过 all-visible 页）：找出死元组的 TID，prune 单页，攒一批死 TID。
2. **扫索引**：从每个索引删除指向这些死 TID 的条目（`ambulkdelete`）。
3. **二次扫堆**：把死元组的 ItemId 回收为 `LP_UNUSED`，更新 FSM 与 VM。

VACUUM 还推进 relfrozenxid（冻结老元组，防 XID 回卷），见 [../transaction/31-clog-slru](../transaction/31-clog-slru.md)。

---

## 5. 三个辅助 fork

heap 关系在物理上由多个 fork 组成（见 [17-smgr-forks](17-smgr-forks.md)）：
- **Main fork**：数据页本身。
- **FSM**（`freespace/`）：每页剩余空闲空间的近似值，`RelationGetBufferForTuple` 据此快速找放置页。
- **VM**（`visibilitymap.c`）：每页 2 bit，`ALL_VISIBLE`（所有元组对所有事务可见 → index-only scan 可免回表、VACUUM 可跳过）和 `ALL_FROZEN`（全部冻结 → anti-wraparound VACUUM 可跳过）。

---

## 6. 设计模式

- **追加 + 多版本（MVCC）**：UPDATE/DELETE 不原地改，靠 `t_xmin/t_xmax` 表达版本可见性，把"并发读写不互斥"建立在"留下旧版本"之上。
- **间接寻址（ItemId）解耦逻辑/物理位置**：元组可在页内移动而 TID 不变，使单页 prune 成为可能。
- **HOT：对索引透明的更新链**：通过"只链首入索引 + 页内 ctid 链"把高频更新的索引维护与空间回收降为单页操作，并刻意**避免调用用户函数**以保证安全。
- **HINT bits 缓存**：把昂贵的 CLOG 查询结果回写到元组 infomask，后续访问直接读位，是"惰性物化判定结果"的优化。
- **职责分层**：heap 只管堆，索引维护交给执行器/索引 AM；可见性独立在 `heapam_visibility.c`；空间/可见性元信息独立成 FSM/VM fork。

---

## 7. 架构编排

```
ModifyTable（执行器）
  ├─ table_tuple_insert → heap_insert            heapam.c:2004
  │     ├─ RelationGetBufferForTuple (FSM 找页)   hio.c:500
  │     ├─ PageAddItem + 设 t_xmin
  │     └─ XLogInsert(xl_heap_insert)
  ├─ table_tuple_update → heap_update            （HOT 判定）
  └─ ExecInsertIndexTuples（非 HOT 才更新索引）   execIndexing.c

后台/手动 VACUUM
  └─ heap_vacuum_rel → lazy_scan_heap            vacuumlazy.c
        ├─ heap_page_prune（单页 HOT prune）       pruneheap.c
        ├─ 索引 ambulkdelete
        └─ 更新 FSM / VM / relfrozenxid
```

---

## 8. 动手探索

```sql
CREATE EXTENSION pageinspect;

-- 看页内 line pointer 与元组头（lp_flags=2 表示 LP_REDIRECT，HOT 链）
SELECT lp, lp_flags, t_ctid, t_xmin, t_xmax, t_infomask::bit(16)
FROM heap_page_items(get_raw_page('t', 0));

-- 观察 HOT vs 非 HOT 更新
UPDATE t SET non_indexed_col = non_indexed_col + 1;   -- HOT（不改索引列）
EXPLAIN (ANALYZE) UPDATE t SET indexed_col = ...;     -- 非 HOT，要更新索引

-- pg_stat 看 HOT 比例
SELECT relname, n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname='t';

-- VM 状态
SELECT * FROM pg_visibility('t'::regclass);
```

---

## 相关模块

- 抽象层：[10-table-am](10-table-am.md)
- 可见性：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
- 页面格式：[15-page-layout](15-page-layout.md)
- 缓冲/预读：[16-buffer-manager](16-buffer-manager.md)、[17-smgr-forks](17-smgr-forks.md)
- WAL：[../transaction/33-wal](../transaction/33-wal.md)
