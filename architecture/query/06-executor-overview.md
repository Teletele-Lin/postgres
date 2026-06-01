# Executor — 执行器总览与火山模型

> 源码目录：`src/backend/executor/`
> 核心入口：`ExecutorStart/Run/Finish/End` — `src/backend/executor/execMain.c`
> 上游：[04-planner-overview](04-planner-overview.md)（PlannedStmt）　下游：客户端 / DestReceiver
> 配套：[07-executor-expr](07-executor-expr.md)（表达式）、[08-executor-nodes](08-executor-nodes.md)（各节点）

---

## 1. 职责

接收优化器产出的 `PlannedStmt`（Plan 树），实际地**产生结果元组**或**执行数据修改**。具体：
- 把静态 Plan 树实例化为运行期的 **PlanState 树**（携带状态、缓冲、表达式编译结果）。
- 用**火山模型（Volcano / 迭代器模型）** 逐行拉取元组。
- 管理执行期资源：快照、打开的关系、TupleTableSlot、内存上下文、参数。
- 把结果交给 `DestReceiver`（发往客户端 / 游标 / COPY / SPI）。

---

## 2. 火山模型（pull-based iterator）

`execProcnode.c` 顶部注释（`src/backend/executor/execProcnode.c:25-71`）用一个例子完整讲清了执行器的运作。计划：

```
            Nest Loop (DEPT.mgr = EMP.name)
            /        \
      Seq Scan      Seq Scan
       DEPT          EMP
   (name="shoe")
```

执行分三相，每相都是**树形分发 + 递归到子节点**：

1. **ExecutorStart → InitPlan → ExecInitNode（根）**：`ExecInitNode` 看到是 NestLoop 就调 `ExecInitNestLoop`，后者再对左右子计划调 `ExecInitNode`，递归到底。产物是一棵与 Plan 树同构的 **PlanState 树**（`execProcnode.c:43-52`）。
2. **ExecutorRun → ExecutePlan → 反复 ExecProcNode（根）**：每次调用 `ExecProcNode(根=NestLoop)` 触发 `ExecNestLoop`，它对子计划调 `ExecProcNode`，子计划是顺序扫描就调 `ExecSeqScan` 返回一行；NestLoop 用子节点返回的列拼出连接行返回（`execProcnode.c:54-62`）。
3. **ExecutorEnd → ExecEndNode（根）**：递归释放每个节点的资源（`execProcnode.c:63-66`）。

**核心思想**：父节点向子节点"要下一行"，子节点按需计算、即拉即用。整棵树共享同一次驱动，天然支持流水线、LIMIT 早停、不必物化全部中间结果。

---

## 3. 核心数据结构

### 3.1 四相接口

`src/include/executor/executor.h`：

```c
void ExecutorStart(QueryDesc *queryDesc, int eflags);   // 初始化 PlanState 树
void ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count);  // 拉取元组
void ExecutorFinish(QueryDesc *queryDesc);              // AFTER 触发器、补完修改
void ExecutorEnd(QueryDesc *queryDesc);                 // 释放资源
```

每个都有 `standard_*` 实现 + 同名 hook（`ExecutorStart_hook` 等，`execMain.c:70-71`），扩展（如 `pg_stat_statements`、`auto_explain`）借此包裹执行。

### 3.2 QueryDesc — 执行的完整上下文

`src/include/executor/execdesc.h`。把"执行一个计划"所需的一切打包：`PlannedStmt *plannedstmt`、源 SQL、`Snapshot`、`ParamListInfo`、目标 `DestReceiver`、以及运行期建立的 `EState *estate` 和 `PlanState *planstate`。

### 3.3 EState — 一次执行的全局状态

`src/include/nodes/execnodes.h`。整棵 PlanState 树共享：

- `es_snapshot` / `es_crosscheck_snapshot`：可见性快照。
- `es_range_table` / `es_rteperminfos`：RTE 与权限信息。
- `es_result_relations`：DML 的目标关系（`ResultRelInfo`，含索引、触发器、约束信息）。
- `es_tupleTable`：本次执行所有 `TupleTableSlot` 的登记处，统一释放。
- `es_query_cxt`：执行期内存上下文。
- `es_param_exec_vals`：InitPlan/相关子查询用的执行期参数。

### 3.4 PlanState — 节点运行期基类

`src/include/nodes/execnodes.h:1207`。每个 Plan 节点对应一个 PlanState：

```c
typedef struct PlanState {
    pg_node_attr(abstract)
    NodeTag     type;
    Plan       *plan;                  // 对应的静态 Plan 节点
    EState     *state;                 // 指回 EState
    ExecProcNodeMtd ExecProcNode;      // 拉取下一行的方法（可被包裹）
    ExecProcNodeMtd ExecProcNodeReal;  // 真实方法（instrument 包裹时保存原值）
    Instrumentation *instrument;       // EXPLAIN ANALYZE 计时计数
    ExprState  *qual;                  // 过滤条件（编译后的表达式）
    PlanState  *lefttree;              // outer 子节点
    PlanState  *righttree;             // inner 子节点
    List       *targetlist;
    TupleTableSlot *ps_ResultTupleSlot;
    ExprContext    *ps_ExprContext;    // 表达式求值上下文（每元组内存域）
    ProjectionInfo *ps_ProjInfo;       // 投影（输出列计算）
    Bitmapset  *chgParam;              // 变化的参数 ID 集合 → 触发 rescan
    ...
} PlanState;
```

派生层次（用首字段嵌入实现"继承"）：`ScanState`（加 `ss_currentRelation`/`ss_ScanTupleSlot`）→ `SeqScanState`；`JoinState`（加 `jointype`）→ `HashJoinState`；`AggState`、`SortState` 等。详见 [08-executor-nodes](08-executor-nodes.md)。

### 3.5 ExecProcNode 的分发与 rescan

`src/include/executor/executor.h:321`：

```c
static inline TupleTableSlot *
ExecProcNode(PlanState *node)
{
    if (node->chgParam != NULL)   // 上层参数变了？
        ExecReScan(node);         // 先重扫（相关子查询、nestloop 内表）
    return node->ExecProcNode(node);  // 调具体节点方法
}
```

`node->ExecProcNode` 是函数指针（`ExecProcNodeMtd`，`execnodes.h:1186`），由 `ExecInitNode` 时设为对应实现（如 `ExecSeqScan`）。`chgParam` 非空意味着外部参数变化，需先 `ExecReScan` 重置子树状态——这正是相关子查询、nestloop 参数化内表的实现机制。

---

## 4. 核心算法：启动与运行

### 4.1 ExecutorStart → InitPlan

`standard_ExecutorStart()`（`execMain.c:143`）建 `EState`、设置快照与范围表，调 `InitPlan()`（`execMain.c:847`）：后者建立结果关系信息（`ResultRelInfo`）、为每个 rowmark 准备、然后 `ExecInitNode(plannedstmt->planTree)` 递归构建整棵 PlanState 树。`eflags`（如 `EXEC_FLAG_EXPLAIN_ONLY`、`EXEC_FLAG_REWIND`）控制初始化行为。

### 4.2 ExecutorRun → ExecutePlan

`standard_ExecutorRun()`（`execMain.c:318`）调 `ExecutePlan()`，后者是火山模型的主循环：反复 `ExecProcNode(根 PlanState)`，每得到一行就交给 `DestReceiver->receiveSlot()`，直到返回空槽或达到 `count`（LIMIT/分批 Execute）。`ScanDirection` 支持游标的向前/向后扫描。

### 4.3 每元组内存管理

每个 PlanState 有 `ps_ExprContext`，其中 `ecxt_per_tuple_memory` 是"每行临时上下文"。节点处理完一行就 `ResetExprContext()` 一次性回收该行产生的所有临时内存（表达式中间结果、函数返回的临时对象）。这是 [../infra/70-memory-context](../infra/70-memory-context.md) 区域式管理在执行器里的典型用法，避免逐对象 free。

---

## 5. TupleTableSlot — 元组容器抽象

`src/include/executor/tuptable.h`。Slot 是执行器里传递一行数据的统一容器，屏蔽底层物理形态：

| Slot 类型 | ops 虚表 | 存储形态 |
|----------|---------|---------|
| Virtual | `tts_virtual_ops` | 仅 `tts_values[]`/`tts_isnull[]`，无物理元组 |
| Heap | `tts_buffer_heap_ops` | 指向 buffer 中的物理 HeapTuple（带 pin） |
| MinimalTuple | `tts_minimal_ops` | MinimalTuple（去系统列的轻量格式，用于排序/哈希落盘） |

`TupleTableSlotOps` 是函数指针虚表（`getsomeattrs`、`copyslot`、`materialize` 等），实现"同一接口、多种物理表示"的多态。`slot_getattr()`/`ExecStoreHeapTuple()`/`ExecMaterializeSlot()` 等通过虚表分发。这让上层节点不关心数据来自磁盘页、虚拟计算还是排序临时文件。

---

## 6. DestReceiver — 结果出口抽象

`src/include/tcop/dest.h`。元组的目的地用虚表 `DestReceiver`（`rStartup`/`receiveSlot`/`rShutdown`/`rDestroy`）抽象：

| Dest | 实现 | 去向 |
|------|------|------|
| `DestRemote` | `printtup.c` | 经 libpq 协议发客户端 |
| `DestTuplestore` | tstoreReceiver.c | 游标 / WITH HOLD |
| `DestCopyOut` | COPY | COPY TO 输出 |
| `DestSPI` | SPI | 存储过程内部接收 |
| `DestIntoRel` | CREATE TABLE AS | 写入新表 |

---

## 7. 设计模式

- **火山迭代器**：统一的 `ExecInitNode/ExecProcNode/ExecEndNode` 三相分发 + 递归，把任意算子组成可流水线执行的树。
- **虚表多态（C 风格 OOP）**：`PlanState.ExecProcNode`（每节点方法指针）、`TupleTableSlotOps`、`DestReceiver`、`ExprState.evalfunc`——全是"函数指针结构体"实现的多态。
- **首字段嵌入式继承**：`SeqScanState ⊃ ScanState ⊃ PlanState`，靠把基类放第一个字段实现安全向上转型。
- **方法包裹（ExecProcNodeReal）**：EXPLAIN ANALYZE 把 `ExecProcNode` 换成计时版 `ExecProcNodeInstr`，原方法存 `ExecProcNodeReal`，对节点逻辑零侵入。
- **区域内存 + 每元组 reset**：用 MemoryContext 生命周期对齐"一行的处理"，把内存管理从手工 free 降维成批量回收。
- **参数驱动 rescan**：`chgParam` + `ExecReScan` 用统一机制表达相关子查询、参数化 nestloop 的"换一组外部值重算"。

---

## 8. 架构编排

```
PortalRun / ProcessQuery
  ├─ ExecutorStart(queryDesc, eflags)          execMain.c:124
  │     └─ standard_ExecutorStart               execMain.c:143
  │           └─ InitPlan → ExecInitNode(根)     execMain.c:847 / execProcnode.c
  │                 → 构建 PlanState 树（递归）
  ├─ ExecutorRun(queryDesc, dir, count)         execMain.c:308
  │     └─ ExecutePlan：循环 ExecProcNode(根)
  │           每行 → DestReceiver->receiveSlot
  ├─ ExecutorFinish                              AFTER 触发器等
  └─ ExecutorEnd → ExecEndNode(根)               释放资源
```

---

## 9. 动手探索

```sql
-- 火山模型的早停：LIMIT 让上层只拉前几行，下层不必算完
EXPLAIN ANALYZE SELECT * FROM big ORDER BY id LIMIT 5;

-- actual rows / loops 揭示每个节点被拉取了多少次（nestloop 内表 loops 大）
EXPLAIN ANALYZE SELECT * FROM a JOIN b ON a.id=b.id;

-- 观察相关子查询的 rescan
EXPLAIN ANALYZE SELECT *, (SELECT count(*) FROM b WHERE b.a_id=a.id) FROM a;
```

调试：在 `ExecProcNode` 或某具体 `ExecSeqScan` 下断点，`bt` 看火山调用栈如何从根逐层下探。

---

## 相关模块

- 表达式：[07-executor-expr](07-executor-expr.md)
- 各算子：[08-executor-nodes](08-executor-nodes.md)
- 数据来源：[../storage/10-table-am](../storage/10-table-am.md)
- 内存：[../infra/70-memory-context](../infra/70-memory-context.md)
