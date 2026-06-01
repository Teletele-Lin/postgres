# Table Access Method — 表访问方法抽象层

> 源码：`src/backend/access/table/`、`src/include/access/tableam.h`
> 高层文档：`doc/src/sgml/tableam.sgml`
> 上游：[../query/08-executor-nodes](../query/08-executor-nodes.md)　下游：[11-heap](11-heap.md)（默认实现）

---

## 1. 职责

Table AM 是执行器与具体表存储格式之间的**抽象层**。它定义"一张表能做什么"（扫描、取行、增删改、加锁、VACUUM、索引回表…），而把"怎么做"留给具体 AM（默认是 `heap`）。引入这层抽象（PostgreSQL 12 起）的目的：让 zheap、列存、压缩存储等替代存储引擎能以扩展形式接入，而执行器代码不变。

默认 AM 名为 `"heap"`（`tableam.h:29`），但执行器**只通过 `TableAmRoutine` 虚表调用**，不直接调 `heap_*`。

---

## 2. 目录结构

```
src/backend/access/table/
  table.c       table_open/table_close 等关系打开包装
  tableam.c     table_beginscan/table_getnextslot/table_insert/... 通用包装；
                GetTableAmRoutine() 校验回调齐全
  tableamapi.c  从 pg_am 的 handler 函数取得 TableAmRoutine
src/include/access/
  tableam.h     TableAmRoutine 虚表定义 + table_* 内联包装函数
  relscan.h     TableScanDesc 等扫描描述符
```

调用链约定：执行器调 `tableam.h` 里的 `table_xxx()` 内联包装 → 包装转发到 `rel->rd_tableam->xxx()`（虚表中的函数指针）→ 具体 AM 实现（heap 的 `heapam_handler.c`）。

---

## 3. 核心数据结构

### 3.1 TableAmRoutine — 函数指针虚表

`src/include/access/tableam.h:321`。一个全是函数指针的大结构体，`type` 字段为 `T_TableAmRoutine`（节点系统打标）。`GetTableAmRoutine()`（`tableam.h:318` 注释）会断言所有**必需**回调都已填充。按功能分组：

```c
typedef struct TableAmRoutine {
    NodeTag type;                        // T_TableAmRoutine

    /* —— Slot —— */
    const TupleTableSlotOps *(*slot_callbacks)(Relation);   // 该 AM 用哪种 slot

    /* —— 表扫描 —— */
    TableScanDesc (*scan_begin)(Relation, Snapshot, int nkeys, ScanKeyData *,
                                ParallelTableScanDesc, uint32 flags);  // tableam.h:360
    void (*scan_end)(TableScanDesc);
    void (*scan_rescan)(TableScanDesc, ScanKeyData *, bool, bool, bool, bool);
    bool (*scan_getnextslot)(TableScanDesc, ScanDirection, TupleTableSlot *);

    /* —— 索引回表 —— */
    struct IndexFetchTableData *(*index_fetch_begin)(Relation);
    bool (*index_fetch_tuple)(struct IndexFetchTableData *, ItemPointer,
                              Snapshot, TupleTableSlot *,
                              bool *call_again, bool *all_dead);        // tableam.h:492

    /* —— DML —— */
    void      (*tuple_insert)(Relation, TupleTableSlot *, CommandId, int options,
                              struct BulkInsertStateData *);            // tableam.h:546
    TM_Result (*tuple_delete)(Relation, ItemPointer, CommandId, Snapshot, ...);
    TM_Result (*tuple_update)(Relation, ItemPointer, TupleTableSlot *, CommandId, ...);
    TM_Result (*tuple_lock)(Relation, ItemPointer, Snapshot, TupleTableSlot *, ...);
    void      (*multi_insert)(Relation, TupleTableSlot **, int nslots, ...);

    /* —— 取单行可见性 —— */
    bool (*tuple_satisfies_snapshot)(Relation, TupleTableSlot *, Snapshot);

    /* —— DDL / 维护 —— */
    void (*relation_set_new_filelocator)(Relation, const RelFileLocator *, ...);
    void (*relation_copy_data)(Relation, const RelFileLocator *);
    void (*relation_copy_for_cluster)(Relation OldHeap, Relation NewHeap, ...);
    void (*relation_vacuum)(Relation, struct VacuumParams *, BufferAccessStrategy);
    void (*scan_analyze_next_block)(...);                 // ANALYZE 抽样
    double (*index_build_range_scan)(...);                // 建索引时全表/范围扫描
    void (*index_validate_scan)(...);
    uint64 (*relation_size)(Relation, ForkNumber);
    bool (*relation_needs_toast_table)(Relation);
    ...
} TableAmRoutine;
```

`scan_begin` 的 `flags`（`tableam.h:353-358`）用 `ScanOptions` 位标识扫描类型（`SO_TYPE_SEQSCAN`/`SO_TYPE_BITMAPSCAN`…）、行为开关（`SO_ALLOW_STRAT` 允许 buffer 访问策略、`SO_ALLOW_SYNC` 允许同步扫描、`SO_ALLOW_PAGEMODE`）、以及快照是否需在 `scan_end` 释放。

### 3.2 TM_Result — DML 结果码

`tuple_update/delete/lock` 返回 `TM_Result`（`tableam.h:145` 附近）：`TM_Ok`、`TM_Invisible`、`TM_SelfModified`、`TM_Updated`（被并发更新）、`TM_Deleted`、`TM_BeingModified`。执行器据此决定是报错、跳过，还是走 EvalPlanQual 重试（见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)）。

### 3.3 TableScanDesc

扫描的运行期句柄，通常被各 AM 嵌进更大的私有结构（heap 的 `HeapScanDescData`）。含关系、快照、扫描键、并行扫描描述符等。

---

## 4. 核心算法：注册与分发

### 4.1 AM 注册（pg_am）

每个 Table AM 在系统表 `pg_am` 中有一行，`amtype='t'`，`amhandler` 指向一个返回 `TableAmRoutine*` 的 C 函数（heap 是 `heap_tableam_handler`）。建表时 `CREATE TABLE ... USING heap` 决定用哪个 AM，记入 `pg_class.relam`。

打开关系时 relcache 调 handler 拿到虚表存进 `RelationData.rd_tableam`（见 [../catalog/41-caches](../catalog/41-caches.md)）。

### 4.2 内联包装层

`tableam.h` 提供大量 `static inline` 包装，统一参数校验/快照管理后转发虚表。例如：

```c
static inline bool
table_scan_getnextslot(TableScanDesc sscan, ScanDirection direction,
                       TupleTableSlot *slot)
{
    ...
    return sscan->rs_rd->rd_tableam->scan_getnextslot(sscan, direction, slot);
}
```

执行器（如 `nodeSeqscan.c`）只见 `table_scan_getnextslot()`，对底层 AM 完全无感。

### 4.3 一次顺序扫描的分发路径

```
ExecSeqScan (nodeSeqscan.c)
  → table_scan_getnextslot()          tableam.h 内联包装
     → rd_tableam->scan_getnextslot   虚表分发
        → heap_getnextslot()          heapam.c（具体实现）
           → heapgettup / ReadBuffer  真正读页
```

---

## 5. 设计模式

- **策略 / 虚表多态**：`TableAmRoutine` 是 C 里用"函数指针结构体 + 注册表"实现的接口。执行器面向接口编程，存储引擎可插拔。
- **抽象层 + 内联包装**：`table_*` 内联函数集中做公共逻辑（快照分配、断言、统计），各 AM 只实现差异部分，减少重复且保持调用点整洁。
- **能力声明（必填 vs 可选回调）**：`GetTableAmRoutine` 断言必需回调齐全；可选回调（如 `multi_insert`）缺失时上层走通用路径。
- **结果码驱动重试（TM_Result）**：把"并发冲突"作为返回值上交执行器，由更高层（EvalPlanQual）决定语义，而非在 AM 内部硬编码隔离级别行为。

---

## 6. 架构编排

```
执行器节点（SeqScan/IndexScan/ModifyTable）
  │ table_beginscan / table_scan_getnextslot / table_tuple_insert ...
  ▼  （tableam.h 内联包装）
rel->rd_tableam : TableAmRoutine*        ← relcache 从 pg_am handler 取得
  │ 虚表分发
  ▼
具体 AM（heap）：heapam_handler.c 把 heap_* 包装成回调
  ▼
heap_*（heapam.c）→ Buffer Manager → SMGR → 磁盘
```

---

## 7. 动手探索

```sql
-- 列出已注册的表访问方法
SELECT amname, amhandler FROM pg_am WHERE amtype='t';

-- 建表时指定/查看 AM
CREATE TABLE t2 (a int) USING heap;
SELECT relname, (SELECT amname FROM pg_am WHERE oid=relam) FROM pg_class WHERE relname='t2';

-- 默认 AM
SHOW default_table_access_method;
```

调试：在 `table_scan_getnextslot` 下断点，`p rel->rd_tableam` 查看虚表；`bt` 观察从执行节点到 `heap_getnextslot` 的分发链。

---

## 相关模块

- 默认实现：[11-heap](11-heap.md)
- 调用方：[../query/08-executor-nodes](../query/08-executor-nodes.md)
- 关系描述符：[../catalog/41-caches](../catalog/41-caches.md)
- 索引侧对称抽象：[12-index-am](12-index-am.md)
