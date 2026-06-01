# ARCHITECTURE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build

This repo uses **Meson** as the primary build system. The build directory is `build/`.

```bash
meson setup build --prefix=$PGHOME -Dcassert=true  -Dbuildtype=debugoptimized -Dc_args="-fno-omit-frame-pointer" -Dcpp_args="-fno-omit-frame-pointer"
ninja -C build install
pg_ctl -D $PGDATA -l $PGDATA/start.log restart
```

First-time init: `rm -rf $PGDATA && initdb -D $PGDATA && pg_ctl -D $PGDATA -l $PGDATA/start.log start`

Run regression tests:
```bash
meson test -C build --suite regress
meson test -C build --suite regress <testname>     # single test
meson test -C build --suite isolation <spec_name>   # isolation test
```

## 总：PostgreSQL 整体架构

PostgreSQL 采用 **多进程架构**（非多线程）。整体分为以下模块：

| 模块 | 源码目录 | 职责 |
|------|----------|------|
| Parser（语法解析） | `src/backend/parser/` | SQL 字符串 → RawStmt 语法树 |
| Analyzer（语义分析） | `src/backend/parser/` | RawStmt → Query 查询树（名称解析、类型检查） |
| Rewriter（重写器） | `src/backend/rewrite/` | 视图展开、规则系统、RLS |
| Planner（优化器） | `src/backend/optimizer/` | Query → Plan（路径枚举、代价估算、join 顺序） |
| Executor（执行器） | `src/backend/executor/` | Plan 树的火山模型执行 |
| Access Methods（访问方法） | `src/backend/access/` | Table AM (heap)、Index AM (btree/gin/gist/brin/hash/spgist) |
| Storage（存储系统） | `src/backend/storage/` | Buffer Manager、Page Layout、File I/O、Lock Manager、IPC |
| Transaction（事务系统） | `src/backend/access/transam/` | XID 管理、CLOG、MVCC 可见性、2PC |
| WAL（预写日志） | `src/backend/access/transam/xlog*.c` | WAL 写入、Checkpoint、Recovery |
| Catalog（系统表） | `src/backend/catalog/`、`src/include/catalog/` | pg_class/pg_attribute 等系统表定义与访问 |
| Postmaster（进程管理） | `src/backend/postmaster/` | 主进程、后台进程（bgwriter/walwriter/checkpointer/autovacuum） |
| Replication（复制） | `src/backend/replication/` | 物理流复制、逻辑解码、发布订阅 |
| Commands（DDL/DML 命令） | `src/backend/commands/` | CREATE/ALTER/DROP/VACUUM/COPY 等具体实现 |
| Nodes（节点系统） | `src/include/nodes/` | 内存中树形数据结构的创建、复制、比较、序列化 |
| Utilities（工具函数） | `src/backend/utils/` | MemoryContext、错误处理、GUC、缓存等 |

**外部接口：**
| 模块 | 目录 | 职责 |
|------|------|------|
| libpq | `src/interfaces/libpq/` | C 客户端库，实现 FE/BE 协议 |
| psql | `src/bin/psql/` | 命令行客户端 |
| pg_dump | `src/bin/pg_dump/` | 逻辑备份 |
| PL Languages | `src/pl/` | plpgsql/plperl/plpython 过程语言 |
| Extensions | `contrib/` | postgres_fdw、pg_stat_statements 等扩展 |

**查询处理流水线（exec_simple_query, `src/backend/tcop/postgres.c:1012`）：**
```
SQL字符串 → [Parser] → RawStmt → [Analyzer] → Query → [Rewriter] → Query → [Planner] → PlannedStmt → [Executor] → 结果元组
```

---

## 分模块详解

### 1. Parser（语法解析）

**目录结构：**
```
src/backend/parser/
  gram.y          - Bison 语法文件（~17K行），定义所有SQL语法规则，产生RawStmt节点
  scan.l          - Flex 词法文件，将SQL字符串切分为token
  scansup.c       - scan.l 辅助函数（处理转义字符串等）
  parser.c        - raw_parser() 入口：调用 lexer + grammar
  gramparse.h     - 语法解析共享定义
  check_keywords.pl - 关键字检查脚本
```

**核心接口（`src/include/parser/parser.h`）：**
```c
List *raw_parser(const char *str, RawParseMode mode);
```
返回 `List *`，每个元素是一个 `RawStmt`（包含 `RawStmt.stmt` 指向具体的 statement node，如 `SelectStmt`、`InsertStmt` 等）。

**parse node 定义：** `src/include/nodes/parsenodes.h`
- `SelectStmt` -- 包含 `targetList`, `fromClause`, `whereClause`, `groupClause`, `sortClause`, `limitCount` 等字段
- `InsertStmt`, `UpdateStmt`, `DeleteStmt`, `CreateStmt`, `AlterTableStmt` 等
- `RawStmt` -- 包装器，包含 `stmt`（具体的 statement node）、`stmt_location`、`stmt_len`

**gram.y 的关键设计原则（注释第28-35行）：** 语法规则**不应访问数据库**也不应依赖可变状态（如SET变量），因为整个多语句字符串会在第一个语句执行前就全部解析完毕。所有需要查表的操作放在后续的 parse analysis 阶段。

---

### 2. Analyzer（语义分析）

**目录结构：**
```
src/backend/parser/
  analyze.c          - parse_analyze() 入口，对每种 statement 调相应的 transform 函数
  parse_clause.c     - 解析 FROM/WHERE/HAVING/GROUP BY/ORDER BY/LIMIT 等子句
  parse_relation.c   - 解析表名、列名，构建 RangeTblEntry (RTE)
  parse_expr.c       - 表达式解析（运算符、CASE、子查询等）
  parse_func.c       - 函数调用解析（名称查找、类型匹配、权限检查）
  parse_oper.c       - 运算符解析
  parse_type.c       - 类型名称解析
  parse_coerce.c     - 类型强制转换（coercion）
  parse_collate.c    - 排序规则处理
  parse_cte.c        - WITH 子句（CTE）处理
  parse_target.c     - SELECT 目标列表处理
  parse_agg.c        - 聚合函数处理（GROUP BY 校验）
  parse_param.c      - $n 参数处理
  parse_node.c       - 节点创建/修改辅助
  parse_merge.c      - MERGE 语句处理
  parse_utilcmd.c    - CREATE/ALTER 等 utility 命令的 transform
  parse_jsontable.c  - JSON_TABLE 处理
```

**核心入口（`src/include/parser/analyze.h`）：**
```c
Query *parse_analyze_fixedparams(RawStmt *parseTree, const char *sourceText,
                                  const Oid *paramTypes, int numParams,
                                  QueryEnvironment *queryEnv);
```
返回单个 `Query` 节点。

**核心数据结构 `Query`（`src/include/nodes/parsenodes.h` 第100行起）：**
```c
typedef struct Query {
    NodeTag    type;
    CmdType    commandType;    // SELECT/INSERT/UPDATE/DELETE/MERGE/UTILITY
    QuerySource querySource;   // 来源：原始SQL、规则展开等
    List      *rtable;         // RangeTblEntry 列表
    FromExpr  *jointree;       // FROM + WHERE 表达式树
    List      *targetList;     // TargetEntry 列表
    List      *groupClause;    // GROUP BY
    List      *sortClause;     // ORDER BY
    Node      *limitOffset;
    Node      *limitCount;
    List      *windowClause;   // WINDOW 子句
    // ... ON CONFLICT, RETURNING, WITH (CTE), locking 等
} Query;
```

**语义分析核心工作：**
- `transformSelectStmt()` → `transformFromClauseItem()` 遍历 FROM 中的每个项目，为每张表打开 relation descriptor，创建 `RangeTblEntry`
- `transformExpr()` 递归地做表达式类型推导、运算符解析、隐式类型转换
- `parse_relation.c` 中的 `addRangeTableEntry()` 负责创建 RTE，记录表名、别名、列信息
- `check_ungrouped_columns_walker()` 检查 GROUP BY 是否覆盖所有非聚合列

**关键结构 `RangeTblEntry`（RTE，`parsenodes.h`）：** 一条 RTE 代表 FROM 中的一个表/子查询/函数调用/join/CTE/values 子句。包含 `relid` (表的 OID)、`eref` (别名)、`coldeflist`、`tablesample` 等信息。

**关键结构 `ParseState`（`src/include/parser/parse_node.h`）：** 语义分析过程中贯穿所有 transform 函数的上下文，保存当前打开的 Relation 列表、namespace 信息、参数类型信息、当前的 RTE 索引等。

---

### 3. Rewriter（重写器）

**目录结构：**
```
src/backend/rewrite/
  rewriteHandler.c  - QueryRewrite()，应用视图规则和 RLS
  rewriteManip.c    - 节点树操作（替换变量、检查 sublink 等）
  rewriteDefine.c   - 创建/删除规则
  rewriteRemove.c   - 删除规则
  rewriteSupport.c  - 规则系统辅助函数
  rewriteSearchCycle.c - WITH RECURSIVE 的循环检测
```

**核心入口（`src/include/rewrite/rewriteHandler.h`）：**
```c
List *QueryRewrite(Query *parsetree);
```
将单个 Query 转换为一个或多个 Query（因为规则可能产生多条语句）。

**视图展开原理：** 当 Query 的 RTE 引用一个视图时，`rewriteHandler.c` 读取 `pg_rewrite` 系统表中的规则。关键函数 `applyRule()`：用规则定义的 substitute Query 替换 RTE，将 RTE 中引用的列映射为视图 Query 的 targetList。经过重写后，视图对外部查询完全透明。

---

### 4. Planner/Optimizer（优化器）

**目录结构：**
```
src/backend/optimizer/
  README                         - 优化器架构说明文档（必须读）
  plan/
    planner.c                    - planner() / standard_planner() 顶层入口
    planmain.c                   - query_planner(): 主join规划循环
    planagg.c                    - 聚合优化（如 MIN/MAX 用索引）
    subselect.c                  - 子查询规划
    createplan.c                 - 将最优 Path 树转换为 Plan 树
  path/
    costsize.c                   - 代价估算函数（cost_seqscan/cost_index/cost_nestloop/cost_mergejoin/cost_hashjoin）
    allpaths.c                   - 为每个基表创建所有可能的扫描路径
    joinpath.c                   - 为每个 join rel 创建所有可能的 join 路径
    joinrels.c                   - make_rel_from_joinlist(): join 顺序枚举（动态规划）
    indxpath.c                   - 索引扫描路径生成
    tidpath.c                    - TID 扫描路径
    equivclass.c                 - 等价类（= 传递闭包），用于推导隐含的 join 条件
    pathkeys.c                   - 排序键（PathKey）处理
  prep/
    prepjointree.c               - join 树 flatten、outer join 重排
    preptlist.c                  - target list 预处理
    prepqual.c                   - WHERE 条件规范化（CNF 转换）
    prepagg.c                    - 聚合预处理
    preptlist.c                  - 预处理 target list
  util/
    relnode.c                    - RelOptInfo 创建与管理
    clausesel.c                  - 选择率估算
    restrictinfo.c               - RestrictInfo 创建与管理
    plancat.c                    - 从系统表获取统计信息（行数、页面数、直方图等）
    var.c                        - Var 节点重映射
    tlist.c                      - TargetEntry 辅助
    placeholder.c                - PlaceHolderVar 处理
  geqo/                          - 遗传算法优化器（表太多时的备选）
```

**核心架构（来自 `src/backend/optimizer/README`）：**

优化器为每个基本关系和 join 关系创建 `RelOptInfo` 结构，给每个 `RelOptInfo` 收集所有可行的访问/join方式的 `Path` 节点。最终选择代价最低的 Path。

**标准 join 枚举算法（动态规划）：**
1. Generate all 2-way join paths
2. Generate all 3-way join paths from 2-way join paths
3. ... continue until all N base relations are joined

三个 join 策略：`NestLoop`（内表每行扫描一次）、`MergeJoin`（两表排序后合并）、`HashJoin`（内表构建哈希表，外表探测）。

**核心数据结构（`src/include/nodes/pathnodes.h`）：**
```c
typedef struct RelOptInfo {
    Relids      relids;            // 该 rel 包含的基表 RT 索引集合
    double      rows;              // 估算行数
    int         width;             // 平均元组宽度（字节）
    List       *pathlist;          // 可行的 Path 列表
    Path       *cheapest_total_path;
    List       *indexlist;         // 该表可用的所有索引 (IndexOptInfo)
    List       *joininfo;          // join 条件 (RestrictInfo)
    // ...分片信息、partition 信息等
} RelOptInfo;

typedef struct Path {
    NodeTag     type;
    RelOptInfo *parent;
    Cost        startup_cost;      // 返回第一行前的启动代价
    Cost        total_cost;        // 返回所有行的总代价
    List       *pathkeys;          // 输出行的排序键
    // ...param_info (参数化路径的外部参数)
} Path;
```
关键 Path 子类型在 `src/include/nodes/pathnodes.h` 中定义：`IndexPath`、`BitmapHeapPath`、`NestPath`、`MergePath`、`HashPath`、`AppendPath` 等。

**代价估算（`src/backend/optimizer/path/costsize.c`）：**
- `cost_seqscan()`：`startup = 0, total = seq_page_cost * relpages + cpu_tuple_cost * reltuples`
- `cost_index()`：考虑 B-tree 遍历代价 + 叶子页扫描代价 + 回表代价
- `cost_nestloop()`：`outer_rescan * (inner_cost per scan)`
- `cost_hashjoin()`：内表扫描 + 哈希表构建 + 外表扫描 + 探测

代价参数：`seq_page_cost` (default 1.0), `random_page_cost` (default 4.0), `cpu_tuple_cost` (0.01), `cpu_operator_cost` (0.0025)。

**`standard_planner()` 函数流程（`src/backend/optimizer/plan/planner.c:321`）：**
1. 创建 `PlannerGlobal`（跨子查询的全局状态）
2. 评估是否允许并行查询 (`max_parallel_hazard()`)
3. 调用 `subquery_planner()` 做单层查询优化：
   - `pull_up_sublinks()` -- 将 EXISTS/IN 子查询提升为 join
   - `preprocess_expression()` -- 表达式预处理（常量折叠、简化布尔表达式等）
   - `inheritance_planner()` 或 `grouping_planner()` -- 主规划
4. 从 `final_rel->cheapest_total_path` 提取最终 Plan，创建 `PlannedStmt`

---

### 5. Executor（执行器）

**目录结构：**
```
src/backend/executor/
  execMain.c           - ExecutorStart/Run/Finish/End 顶层接口
  execProcnode.c       - ExecProcNode/ExecInitNode/ExecEndNode 分发
  execExpr.c           - 表达式编译与执行 (ExprState/ExprEvalStep)
  execExprInterp.c     - 表达式解释器（对 ExprEvalStep 的 JIT-friendly 循环）
  execTuples.c         - TupleTableSlot 管理
  execScan.c           - 通用扫描框架（ExecScan, ExecAssignScanProjectionInfo）
  execUtils.c          - EState/ExprContext/ResultRelInfo 管理
  execGrouping.c       - 分组聚合哈希表
  execIndexing.c       - 索引插入/删除操作
  execReplication.c    - 逻辑复制相关
  execCurrent.c        - WHERE CURRENT OF 支持
  execJunk.c           - junk 列过滤（用于 RETURNING 等）
  execSRF.c            - 集合返回函数（SRF）
  execPartition.c      - 分区表执行
  execAsync.c          - 异步执行（Foreign Scan）
  nodeSeqscan.c        - SeqScan 执行节点
  nodeIndexscan.c      - IndexScan 执行节点
  nodeIndexonlyscan.c  - IndexOnlyScan 执行节点
  nodeBitmapHeapscan.c - BitmapHeapScan
  nodeHashjoin.c       - HashJoin 执行节点（Hybrid Hash Join 算法）
  nodeHash.c           - Hash 表构建
  nodeMergejoin.c      - MergeJoin 执行节点
  nodeNestloop.c       - NestedLoop 执行节点
  nodeAgg.c            - Aggregation（HashAgg、SortAgg、MixedAgg）
  nodeWindowAgg.c      - Window Function
  nodeSort.c           - Sort 节点
  nodeMaterial.c       - Materialize 节点
  nodeLimit.c          - Limit 节点
  nodeAppend.c         - Append (UNION ALL) 节点
  nodeMergeAppend.c    - MergeAppend 节点
  nodeGather.c         - Gather (并行查询收集) 节点
  nodeModifyTable.c    - INSERT/UPDATE/DELETE/MERGE 执行节点
  nodeSetOp.c          - INTERSECT/EXCEPT
  nodeSubplan.c        - 子查询执行
  nodeUnique.c         - DISTINCT (Sort+Unique or Hash)
  nodeMemoize.c        - Nested Loop 缓存
  nodeLockRows.c       - SELECT FOR UPDATE/SHARE
  nodeForeignscan.c    - Foreign Scan
  nodeCustom.c         - Custom Scan
  nodeCtescan.c        - CTE Scan
  nodeNamedtuplestorescan.c
  nodeFunctionscan.c
  nodeValuesscan.c
  nodeTableFuncscan.c
  nodeResult.c
  nodeWorktablescan.c
  nodeTidrangescan.c
  nodeIncrementalSort.c
  tstoreReceiver.c     - TupleStore DestReceiver
```

**执行器使用火山模型（Volcano-style pull model）**：父节点调用子节点的 `ExecProcNode()` 来拉取下一条元组。

**核心接口（`src/include/executor/executor.h`）：**
```c
void ExecutorStart(QueryDesc *queryDesc, int eflags);
void ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count);
void ExecutorFinish(QueryDesc *queryDesc);
void ExecutorEnd(QueryDesc *queryDesc);
```

**`ExecProcNode` 函数指针（`executor.h:309-317`）：**
```c
static inline TupleTableSlot * ExecProcNode(PlanState *node) {
    if (node->chgParam != NULL)  // 参数变化需要重新扫描
        ExecReScan(node);
    return node->ExecProcNode(node);
}
```
每个 Plan 节点有对应的 PlanState，如 `SeqScanState.execProcNode` 指向 `ExecSeqScan`。

**`QueryDesc`（`src/include/executor/execdesc.h`）：** 执行器的完整上下文——包含 `PlannedStmt`、快照、参数、目标 DestReceiver、以及运行时创建的 EState 和 PlanState 树。

**`EState`（`src/include/nodes/execnodes.h`）：** 一次查询执行的全局状态：`es_range_table`（RTE 列表）、`es_snapshot`、`es_result_relations`（DML 的结果关系列表）、`es_tupleTable`（所有 TupleTableSlot 的链表）。

**`PlanState` 层级（`src/include/nodes/execnodes.h`）：**
- 基类 `PlanState`：`Plan *plan`, `EState *state`, `ExecProcNodeMtd ExecProcNode`, `List *targetlist`, `ExprState *qual`
- `ScanState`（继承 PlanState）：`Relation ss_currentRelation`, `TableScanDesc ss_currentScanDesc`, `TupleTableSlot *ss_ScanTupleSlot`
- `SeqScanState`（继承 ScanState）：增加 `ReadStream *ss_read_stream`用于预读
- `JoinState`（继承 PlanState）：`JoinType jointype`, `PlanState *outerPlanState/innnerPlanState`, join qual
- `HashJoinState`（继承 JoinState）：`HashJoinTable hj_HashTable`, `HashState *hj_HashState`
- `AggState`：聚合哈希表、phase 信息、排序信息

**表达式执行系统：** 表达式在执行前被"编译"为 `ExprState`（包含 `ExprEvalStep` 指令数组）。`ExecInitExpr()` 遍历表达式树产生 `ExprEvalStep` 列表（opcode 如 `EEOP_FUNCEXPR_STRICT`, `EEOP_ASSIGN_TMP`, `EEOP_QUAL` 等）。运行时 `ExecInterpExpr()` 用一个巨大的 switch 循环解释执行这些步骤（位于 `src/backend/executor/execExprInterp.c`，有 JIT 编译优化支持）。

**表达式计算入口（`executor.h:388-395`）：**
```c
static inline Datum ExecEvalExpr(ExprState *state, ExprContext *econtext, bool *isNull) {
    return state->evalfunc(state, econtext, isNull);
}
```
`evalfunc` 在纯解释模式下指向 `ExecInterpExpr`；在 JIT 模式下指向编译好的 native 函数。

**HashJoin 实现（`src/backend/executor/nodeHashjoin.c`）：** 使用 Hybrid Hash Join 算法。第一阶段：扫描内表构建哈希表（`ExecHashJoinImpl`）。内存不足时溢出到磁盘 batch files。对于并行 Hash Join，通过 Barrier IPC 同步各 worker 的构建阶段。

**Agg 实现（`src/backend/executor/nodeAgg.c`）：** 支持 HashAgg（`AGG_HASHED`）、SortAgg（`AGG_SORTED`）、MixedAgg（`AGG_MIXED`）。聚合执行流程：`transvalue = initcond` → 对每条输入执行 `transvalue = transfunc(transvalue, input)` → `result = finalfunc(transvalue)`。支持 partial aggregation（`AggSplit`），用于并行聚合：Partial Agg → Gather → Final Agg。

**TupleTableSlot（`src/include/executor/tuptable.h`）：** 执行器中的元组容器。支持多种存储方式：
- Virtual Slot：`tts_values`/`tts_isnull` 数组，不关联物理存储
- Heap Slot：`tts_tuple` 指向 buffer 中的物理元组
- Minimal Tuple Slot：使用 MinimalTuple（去掉系统列的轻量元组格式）

Slot 操作由 `TupleTableSlotOps` 虚函数表分发：`tts_buffer_heap_ops`, `tts_virtual_ops`, `tts_minimal_tuple_ops`。

**DestReceiver（`src/include/tcop/dest.h`）：** 元组输出目的地抽象：
- `DestRemote` → `printtup.c` 通过 libpq 协议发送给客户端
- `DestSPI` → SPI 接口内部使用
- `DestTuplestore` → 存储到 Tuplestore（游标）
- `DestCopyOut` → COPY 输出
- `DestFunction` → SQL 函数返回集

**DML 执行（`src/backend/executor/nodeModifyTable.c`）：** INSERT/UPDATE/DELETE/MERGE 通过 `ExecModifyTable()` 实现。包含：触发器调用（行级 / 语句级）、外键约束检查、分区路由、RETURNING 处理、UPDATE 的 EvalPlanQual 重检查。

---

### 6. Access Methods（访问方法）

#### 6.1 Table Access Method

**目录结构：**
```
src/backend/access/
  table/              - Table AM 通用框架
    table.c           - table_beginscan/table_getnext_slot/table_insert/table_delete/table_update
    tableam.c         - TableAmRoutine 注册与查找
  heap/               - 默认 Heap Table AM 实现
    heapam.c          - heap_* 函数实现
    heapam_handler.c  - 将 heap_* 包装成 TableAmRoutine 回调
    heapam_visibility.c - 可见性判断
    hio.c             - Heap Insert Or Update（查找合适的页面）
    pruneheap.c       - HOT prune
    vacuumlazy.c      - Lazy VACUUM
    rewriteheap.c     - CLUSTER/VACUUM FULL 的表重写
```

**核心抽象 `TableAmRoutine`（`src/include/access/tableam.h`，约 200+ 行）：**
```c
typedef struct TableAmRoutine {
    // 元组可见性
    bool (*tuple_satisfies_snapshot)(Relation, TupleTableSlot *, Snapshot);
    // 扫描
    TableScanDesc (*scan_begin)(Relation, Snapshot, int nkeys, ScanKey, ...);
    void (*scan_end)(TableScanDesc);
    bool (*scan_getnextslot)(TableScanDesc, ScanDirection, TupleTableSlot *);
    // DML
    void (*tuple_insert)(Relation, TupleTableSlot *, CommandId, int options, BulkInsertState);
    TM_Result (*tuple_delete)(Relation, ItemPointer, CommandId, Snapshot, ...);
    TM_Result (*tuple_update)(Relation, ItemPointer, TupleTableSlot *, CommandId, ...);
    TM_Result (*tuple_lock)(Relation, ItemPointer, ..., LockTupleMode, ...);
    // 索引操作
    bool (*index_fetch_tuple)(..., TupleTableSlot *, bool *call_again, bool *all_dead);
    void (*index_delete_tuples)(Relation, TM_IndexDeleteOp *);
    // 批量插入
    void (*multi_insert)(Relation, TupleTableSlot **, int nslots, CommandId, int options, BulkInsertState);
    // 维护
    void (*relation_set_new_filenode)(Relation, ...);
    bool (*vacuum_rel)(Relation, VacuumParams *, BufferAccessStrategy);
    bool (*analyze_rel)(Relation, VacuumParams *, ...);
    // 等等
} TableAmRoutine;
```

**Heap AM 实现（`src/backend/access/heap/heapam.c`）：**
- `heap_insert()`: 在合适页面找到空闲空间（通过 FSM），放置元组，写 WAL（`xl_heap_insert`），更新索引
- `heap_delete()`: 标记元组的 `t_xmax` = 当前 XID，写 WAL，更新索引
- `heap_update()`: 在 old tuple 上标记 `t_xmax`，在新位置插入 new tuple，写 WAL，处理 HOT 更新（如索引列未变则仅更新 heap）
- `heap_beginscan()`: 创建 `HeapScanDesc`，初始化 ReadStream（用于预读）
- `heap_getnextslot()`: 遍历页面获取下一条可见元组

**MVCC 可见性判断（`src/backend/access/heap/heapam_visibility.c`）：** 核心函数 `HeapTupleSatisfiesMVCC(Relation, HeapTuple, Snapshot, Buffer)`。对于每条元组：
1. 检查 `t_xmin` — 如果插入事务已提交且快照可见 → 继续
2. 检查 `t_xmax` — 如果删除事务已提交且在快照之前 → 不可见
3. 检查 HINT bits (`HEAP_XMIN_COMMITTED`, `HEAP_XMAX_INVALID` 等) 避免重复查 CLOG

#### 6.2 Index Access Methods

**目录结构：**
```
src/backend/access/
  index/              - 通用 Index AM 框架 (genam.c, indexam.c)
  nbtree/             - B-tree (Lehman-Yao 高并发)
  gin/                - GIN (Generalized Inverted Index)
  gist/               - GiST (Generalized Search Tree)
  brin/               - BRIN (Block Range Index)
  hash/               - Hash Index
  spgist/             - SP-GiST (Space-Partitioned GiST)
```

**核心抽象 `IndexAmRoutine`（`src/include/access/amapi.h`）：**
```c
typedef struct IndexAmRoutine {
    IndexBuildResult *(*ambuild)(Relation heap, Relation index, IndexInfo *);
    void (*ambuildempty)(Relation index);
    bool (*aminsert)(Relation, Datum *values, bool *isnull, ItemPointer, ...);
    IndexBulkDeleteResult *(*ambulkdelete)(..., IndexBulkDeleteCallback, ...);
    IndexBulkDeleteResult *(*amvacuumcleanup)(..., IndexVacuumInfo *, ...);
    bool (*amgettuple)(IndexScanDesc, ScanDirection);
    int64 (*amgetbitmap)(IndexScanDesc, TIDBitmap *);
    // ...
} IndexAmRoutine;
```

**B-tree（`src/backend/access/nbtree/`）** — 最重要的 Index AM：
- Lehman-Yao 高并发协议：页面分裂时保持从旧页面到新页面的 right link
- `BTPageOpaqueData`（`src/include/access/nbtree.h:63`）：存储在每页 special area：
  ```c
  BlockNumber btpo_prev;   // 左兄弟
  BlockNumber btpo_next;   // 右兄弟（Lehman-Yao key）
  uint32 btpo_level;       // 树层级（0 = leaf）
  uint16 btpo_flags;       // BTP_LEAF / BTP_ROOT / BTP_DELETED / BTP_META
  BTCycleId btpo_cycleid;  // vacuum cycle ID（检测并发分裂）
  ```
- 搜索时可能因并发分裂落到错误的页，通过 `btpo_next` 向右移动找到正确的页面
- INSERT 可能触发页面分裂，分裂时先创建右页（链接到原页的 next），再原子地插入父指针

**GIN（`src/backend/access/gin/`）：** 倒排索引。适用于数组、全文搜索向量等复合类型。索引结构：entries tree（key → posting list）+ posting tree（每个 key 的 TID 列表）。

**GiST（`src/backend/access/gist/`）：** 通用搜索树。支持自定义数据类型和操作符。平衡树，每个节点存储一个 predicate，查询时剪枝不满足 predicate 的子树。

**BRIN（`src/backend/access/brin/`）：** Block Range Index。对每 N 个连续页面存储 min/max 摘要。体积极小，适合大表自然有序的场景。

---

### 7. Storage System（存储系统）

#### 7.1 Page Layout (bufpage)

**目录：** `src/backend/storage/page/`, `src/include/storage/bufpage.h`

**页面格式（`src/include/storage/bufpage.h:26-80`）：**
```
PageHeaderData | linp1 linp2 linp3 ... (ItemId 数组)
               | ... linpN
               pd_lower ------→           ←------ pd_upper
               | tupleN ...

               | tuple3 tuple2 tuple1 | "special space" |
                                       pd_special ------→
```

**`PageHeaderData` 结构：**
```c
typedef struct PageHeaderData {
    PageXLogRecPtr pd_lsn;     // 最后一次修改此页的WAL LSN
    uint16 pd_checksum;        // 页校验和
    uint16 pd_flags;           // 标志位
    LocationIndex pd_lower;    // ItemId 数组末尾偏移
    LocationIndex pd_upper;    // 元组数据起始偏移
    LocationIndex pd_special;  // special space 起始
    uint16 pd_pagesize_version;
    TransactionId pd_prune_xid; // 最近 prune 的 XID
    ItemIdData pd_linp[FLEXIBLE_ARRAY]; // line pointer 数组
} PageHeaderData;
```

**ItemId（线指针）：** `lp_off`（元组字节偏移）、`lp_flags`（`LP_UNUSED`/`LP_NORMAL`/`LP_REDIRECT`/`LP_DEAD`）、`lp_len`（元组长度）。线指针的间接性使得元组可以在页面内物理移动而不改变逻辑 offset。

**`PageInit()`（`src/backend/storage/page/bufpage.c:41`）：** 用零填充整个页面，设置 `pd_lower = SizeOfPageHeaderData`，`pd_upper = pageSize - specialSize`。

**元组头 `HeapTupleHeaderData`（`src/include/access/htup_details.h`）：**
```c
struct HeapTupleHeaderData {
    TransactionId t_xmin;   // 插入此元组的XID
    TransactionId t_xmax;   // 删除/锁定此元组的XID (0 = still live)
    CommandId t_cid;        // 插入命令ID
    ItemPointerData t_ctid; // 当前TID（block, offset），HOT链 / 更新链
    // t_infomask2 / t_infomask: HINT bits，如 HEAP_XMIN_COMMITTED,
    //   HEAP_XMAX_INVALID, HEAP_HASNULL, HEAP_HASVARWIDTH 等
};
```

HOT（Heap-Only Tuple）更新：如果 UPDATE 没有修改索引列，新元组放在同一页面，旧元组的线指针标记为 `LP_REDIRECT` 指向新元组。索引只指向链首。

#### 7.2 Buffer Manager

**目录：** `src/backend/storage/buffer/`, `src/include/storage/bufmgr.h`, `src/include/storage/buf_internals.h`

**核心接口：**
```c
Buffer ReadBuffer(Relation reln, BlockNumber blockNum);        // pin 一个页面
Buffer ReadBufferExtended(Relation, ForkNumber, BlockNumber, ReadBufferMode, BufferAccessStrategy);
void ReleaseBuffer(Buffer buffer);                             // unpin
void MarkBufferDirty(Buffer buffer);                           // 标记为脏
```

**`BufferAccessStrategyType`（`bufmgr.h:34`）：** `BAS_NORMAL`, `BAS_BULKREAD`, `BAS_BULKWRITE`, `BAS_VACUUM`。每种策略维护一个小的 buffer ring，防止顺序扫描污染整个缓冲池。

**Buffer 描述符（`src/include/storage/buf_internals.h`）：**
```c
typedef struct BufferDesc {
    BufferTag   tag;            // (spcOid, dbOid, relNumber, forkNum, blockNum)
    pg_atomic_uint32 state;     // refcount(18b) + usage_count(4b) + flags(10b)
    int         buf_id;         // 在缓冲池数组中的索引
    ConditionVariable wait_backend_pin;  // 等待pin的条件变量
    // ...
} BufferDesc;
```

**状态位（`buf_internals.h:68-78`）：**
- `BM_LOCKED` — buffer header 已锁
- `BM_DIRTY` — 数据需要写入磁盘
- `BM_VALID` — 数据有效
- `BM_TAG_VALID` — tag 已分配
- `BM_IO_IN_PROGRESS` — I/O 正在进行
- `BM_PIN_COUNT_WAITER` — 有其他进程在等待 sole pin

**Buffer Tag（`buf_internals.h:106`）：** `(spcOid, dbOid, relNumber, forkNum, blockNum)` — 唯一标识一个磁盘页面。Lookup 使用哈希表（`BufTableLookup`），按 `NUM_BUFFER_PARTITIONS`（128）分区以减少锁竞争。

**Clock-sweep 替换策略（`src/backend/storage/buffer/freelist.c`）：** 基于 usage_count 的时钟算法。`usage_count` 最大值为 `BM_MAX_USAGE_COUNT`（5）。扫描时递减 usage_count，选择 usage_count 为 0 且 pin count 为 0 的页面作为淘汰候选。`BM_PERMANENT` 标志保护共享关系。

**`ReadBufferExtended` 流程：**
1. 计算 `BufferTag`
2. `BufTableLookup()` 在哈希表中查找
3. 命中 → pin 缓冲区 → 返回
4. 未命中 → 调用 `StrategyGetBuffer()` 获取空闲 buffer → 将旧页写回（如脏）→ `smgrread()` 读入新页 → 更新哈希表 → 返回

**ReadStream（`src/include/storage/read_stream.h`）：** 顺序扫描的预读框架。
```c
ReadStream *read_stream_begin_relation(int flags, BufferAccessStrategy strategy,
    Relation rel, ForkNumber forknum,
    ReadStreamBlockNumberCB callback, void *callback_private_data, ...);
Buffer read_stream_next_buffer(ReadStream *stream, void **per_buffer_data);
```
`READ_STREAM_SEQUENTIAL` 禁用显式预读提示（让内核检测顺序访问）。
`READ_STREAM_MAINTENANCE` 使用 `maintenance_io_concurrency` 而非 `effective_io_concurrency`。

#### 7.3 SMGR (Storage Manager)

**目录：** `src/backend/storage/smgr/`, `src/include/storage/smgr.h`

**核心：** `SMgrRelation` 表示一个打开的物理关系。接口：
```c
void smgrread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, void *buffer);
void smgrwrite(SMgrRelation, ForkNumber, BlockNumber, const void *buffer, bool skipFsync);
void smgrextend(SMgrRelation, ForkNumber, BlockNumber, const void *buffer, bool skipFsync);
void smgrtruncate(SMgrRelation, ForkNumber, BlockNumber nblocks);
```

底层调用 `md.c`（magnetic disk），使用 POSIX `pread/pwrite` 操作文件。一个 relation 的存储由多个 fork 组成：
- **Main fork**（`MAIN_FORKNUM`）：表/索引数据
- **FSM fork**（`FSM_FORKNUM`）：Free Space Map，快速定位有空闲页的页面
- **Visibility Map fork**（`VISIBILITY_MAP_FORKNUM`）：每页 2 bit，标记 "all-visible" 和 "all-frozen"（用于 index-only scan 和 VACUUM 跳过）

**AIO（`src/backend/storage/aio/`）：** 异步 I/O 子系统。使用 `pgaio_submit()` 提交 I/O 请求，`pgaio_io_wait()` 等待完成。

#### 7.4 Lock Manager

PostgreSQL 有四层锁机制，从底层到高层：

**1. Spinlocks（`src/include/storage/s_lock.h`, `src/include/storage/spin.h`）：**
- 硬件级 TAS（test-and-set）原语
- 仅用于保护极短的临界区（如 LWLock 内部状态）
- 平台相关汇编实现

**2. LWLocks（`src/backend/storage/lmgr/lwlock.c`, `src/include/storage/lwlock.h`）：**
- 读写锁，FIFO 排队
- Wait-free 共享锁获取（CAS 循环，见 `lwlock.c:38-76` 注释）
- 三阶段加锁：Phase 1 (CAS) → Phase 2 (排队) → Phase 3 (再次 CAS) → Phase 4 (sleep)
- `LWLock` 结构：`uint16 tranche`, `pg_atomic_uint32 state`（exclusive sentinel 或 shared count）, `proclist_head waiters`
- 每个 LWLock 填充到 `PG_CACHE_LINE_SIZE`（64B）避免 false sharing
- 关键锁分区：`WALInsertLocks`（128）、`BufferMappingLocks`（128）、`LockManagerLWLocks`（16）、`ProcArrayLock`
- 命名 tranche 通过 `LWLockRegisterTranche()` 注册

**3. Heavyweight Locks（`src/backend/storage/lmgr/lock.c`, `src/include/storage/lock.h`）：**
- SQL 级锁，支持死锁检测
- Lock modes（从低到高）：`AccessShareLock` (SELECT) < `RowShareLock` (SELECT FOR UPDATE) < `RowExclusiveLock` (DML) < `ShareLock` < `ExclusiveLock` < `AccessExclusiveLock` (DDL)
- Lock 对象通过 `LOCKTAG` 哈希定位，持有者和等待者链表
- 死锁检测（`src/backend/storage/lmgr/deadlock.c`）：构建 waits-for 图，用 DFS 检测环

**4. Predicate Locks（`src/backend/storage/lmgr/predicate.c`）：**
- 用于 Serializable Snapshot Isolation（SSI）
- 追踪 SerializableXact (SXACT) 之间的 rw-conflict
- 实现 `README-SSI` 中描述的算法

**Latches（`src/include/storage/latch.h`）：** `Latch` 是进程间等待/唤醒原语。`WaitLatch(latch, events)` 使进程休眠，`SetLatch(latch)` 唤醒等待者。广泛用于 backend 之间的信令（postmaster → backend、checkpointer wakeup 等）。

**PGPROC（`src/include/storage/proc.h`）：** 每个 backend 在共享内存中的 per-process 结构。包含：进程的 XID、LWLock 等待位置、持有/等待的锁信息、latch、subxid 缓存等。

---

### 8. Transaction System（事务系统）

**目录结构：**
```
src/backend/access/transam/
  xact.c           - 事务状态机主逻辑（Start/Commit/Abort/Savepoint）
  transam.c        - TransactionId 分配与 FullTransactionId 管理
  varsup.c         - XID 推进、anti-wraparound VACUUM 触发
  clog.c           - Commit Log（事务状态 2 bits/XID）
  subtrans.c       - 子事务 parent 映射
  multixact.c      - 共享行锁的多事务管理
  twophase.c       - Two-Phase Commit (PREPARE TRANSACTION)
  slru.c           - SLRU (Simple LRU) 通用实现（clog/subtrans/multixact 的基础）
  rmgr.c           - 资源管理器注册表（RmgrTable）
```

**XID 体系（`src/include/access/transam.h`）：**
```c
typedef uint32 TransactionId;    // 32-bit, wraparound
// Special values:
//   InvalidTransactionId      = 0
//   BootstrapTransactionId    = 1
//   FrozenTransactionId       = 2
//   FirstNormalTransactionId  = 3

typedef struct FullTransactionId { uint64 value; }  // 64-bit (epoch32 + xid32)
```

**事务状态机（`src/backend/access/transam/xact.c`）：**
```
TBLOCK_DEFAULT → TBLOCK_STARTED → TBLOCK_INPROGRESS → committed/aborted
```
关键函数：`StartTransactionCommand()`, `CommitTransactionCommand()`, `AbortCurrentTransaction()`。支持隐式事务（单语句）、显式块（BEGIN/COMMIT）、savepoint（`DefineSavepoint/RollbackToSavepoint`）。

**CLOG（Commit Log，`src/backend/access/transam/clog.c`）：** 每个 XID 分配 2 bit 记录其状态：`TRANSACTION_STATUS_IN_PROGRESS` (0), `COMMITTED` (1), `ABORTED` (2), `SUB_COMMITTED` (3)。存储在 `pg_xact/` 下的 SLRU 段文件中。核心查询接口：
```c
TransactionIdGetStatus(xid, &xid);
// 通过 hint bits (t_infomask) 缓存，避免重复查询 CLOG
```

**Subtrans（子事务，`src/backend/access/transam/subtrans.c`）：** 映射子事务 XID → 父事务 XID。同样使用 SLRU，存储在 `pg_subtrans/` 下。

**MultiXact（`src/backend/access/transam/multixact.c`）：** 当多个事务对同一行加共享锁时，分配一个 MultiXactId 代表这个事务集合。存储在 `pg_multixact/` 下，分为 members 文件和 offsets 文件。

**Two-Phase Commit（`src/backend/access/transam/twophase.c`）：** PREPARE TRANSACTION 将事务状态序列化到 `pg_twopase/` 下的文件中，包含：锁信息、XID、GID（全局事务标识符）、通知等。恢复时 `RecoverPreparedTransactions()` 读取这些文件重新获取锁。

---

### 9. MVCC (Multiversion Concurrency Control)

**快照（`src/include/utils/snapshot.h`）：**
```c
typedef struct SnapshotData {
    SnapshotType snapshot_type;     // SNAPSHOT_MVCC / SNAPSHOT_SELF / SNAPSHOT_ANY / ...
    TransactionId xmin;            // XID < xmin 的元组总是可见
    TransactionId xmax;            // XID >= xmax 的元组总是不可见（"未来"）
    TransactionId *xip;            // 快照时刻正在运行的 XID 列表
    uint32 xcnt;                   // xip 数组大小
    TransactionId *subxip;         // 子事务 XID 列表
    int32 subxcnt;
    CommandId curcid;              // 当前事务内命令ID，< curcid 的可见
    bool takenDuringRecovery;       // hot standby 下获取的快照
} SnapshotData;
```

- `SNAPSHOT_MVCC`：正常快照。GetTransactionSnapshot() → GetSnapshotData() 读取共享内存中的 ProcArray，收集所有正在运行的 XID
- `SNAPSHOT_DIRTY`：VACUUM 使用，能看到未提交数据
- `SNAPSHOT_HISTORIC_MVCC`：逻辑解码使用
- `SNAPSHOT_NON_VACUUMABLE`：VACUUM 自己使用，xmin 作为 horizon

**隔离级别（`src/include/access/xact.h:36-39`）：**
```c
#define XACT_READ_COMMITTED   1   // 每个语句一个新快照
#define XACT_REPEATABLE_READ  2   // 整个事务一个快照
#define XACT_SERIALIZABLE     3   // 快照 + 谓词锁 (SSI)
```

---

### 10. WAL (Write-Ahead Log)

**目录结构：**
```
src/backend/access/transam/
  xlog.c            - WAL 主逻辑（insert/fsync/archiving）、WAL writer
  xloginsert.c      - XLogBeginInsert / XLogRegisterData / XLogRegisterBlock / XLogInsert
  xlogreader.c      - WAL 记录解析器（用于 recovery 和逻辑解码）
  xlogrecovery.c    - Crash/Archive Recovery 逻辑
  xlogarchive.c     - WAL archiving (archive_command)
  xlogfuncs.c       - pg_wal_* SQL 函数
  xlogstats.c       - WAL 统计
  xlogutils.c       - 恢复期间的 buffer 管理
  xlogbackup.c      - Backup label 文件
  xlogprefetcher.c  - Recovery 期间预取 WAL 记录
```

**WAL 记录格式（`src/include/access/xlogrecord.h:41-53`）：**
```c
typedef struct XLogRecord {
    uint32       xl_tot_len;   // 整条记录的总长度
    TransactionId xl_xid;      // 所属事务的 XID
    XLogRecPtr   xl_prev;      // 指向前一条记录的指针
    uint8        xl_info;      // 低4位：XLOG内部标志；高4位：RMGR自定义
    RmgrId       xl_rmid;      // 资源管理器 ID (RM_HEAP_ID, RM_BTREE_ID, ...)
    pg_crc32c    xl_crc;       // CRC32 校验和
    // 随后是 XLogRecordBlockHeader(s) 和 主数据
} XLogRecord;
```

**WAL 写入流程（`src/backend/access/transam/xloginsert.c`）：**
1. `XLogBeginInsert()` — 初始化线程局部注册缓冲区
2. `XLogRegisterData(data, len)` — 注册不关联页面的数据
3. `XLogRegisterBlock(block_id, rlocator, forkno, blocknum, page, flags)` — 注册页面引用（包括 full-page image 或 delta）
4. `XLogInsert(RMGR_ID)` — 组装记录、计算 CRC、获取 `WALInsertLock`、复制到 WAL buffer、更新 `CurrBytePos`
5. 如果记录超过 `wal_writer_flush_after` 或者事务提交，调用 `XLogFlush()`

**Resource Managers（`src/include/access/rmgr.h`）：** `RM_XLOG_ID`, `RM_XACT_ID`, `RM_HEAP_ID`, `RM_BTREE_ID`, `RM_GIN_ID`, `RM_GIST_ID` 等。每个 RMGR 实现：
- `rm_redo(xlogreader)` — 重做（恢复时调用）
- `rm_decode(xlogreader)` — 逻辑解码
- `rm_desc(StringInfo, xlogreader)` — 人类可读的描述（pg_waldump 使用）

**Checkpoint（`src/backend/postmaster/checkpointer.c`）：**
执行流程：
1. 获取 `CheckpointLock`（排他）
2. 计算 redo point（所有 dirty page 中最小的 LSN）
3. Flush 所有 WAL 到磁盘
4. 写 checkpoint 记录到 WAL
5. 更新 `pg_control` 文件（checkpoint LSN 等）
6. 等待所有 dirty buffer 写回磁盘

**Recovery（`src/backend/access/transam/xlogrecovery.c`）：**
1. 从 `pg_control` 读取上次 checkpoint 的 redo LSN
2. 从 redo LSN 开始顺序扫描 WAL
3. `XLogReader` 解析每条记录
4. 对于每个 block_id，检查目标页面是否需要回放（redo 规则：如果页面 LSN >= 记录 LSN，则跳过）
5. 调用 `RmgrTable[record->xl_rmid].rm_redo()` 应用变更

---

### 11. Catalog System（系统表）

**目录结构：**
```
src/include/catalog/
  pg_class.h       - 关系（表/索引/视图等）定义: CATALOG(pg_class,1259)
  pg_attribute.h   - 列定义
  pg_type.h        - 数据类型定义
  pg_proc.h        - 函数/过程定义
  pg_index.h       - 索引定义
  pg_am.h          - 访问方法 (heap/btree/gist/...)
  pg_namespace.h   - Schema
  pg_database.h    - 数据库
  pg_authid.h      - 角色/用户
  pg_*.dat         - 初始数据（initdb 时加载）
  genbki.h         - BKI (Backend Interface) 宏定义
src/backend/catalog/
  heap.c           - heap_create / heap_create_with_catalog
  indexing.c       - index_create
  catalog.c        - IsSystemCatalog 等辅助
  namespace.c      - schema/namespace 查找
  pg_*.c           - 各系统表的操作（pg_class, pg_proc, pg_type, ...）
src/backend/utils/cache/
  relcache.c       - Relation 描述符缓存 (RelationData)
  catcache.c       - 系统表元组缓存 (按 hash key)
  syscache.c       - 在 catcache 之上的 thin wrapper
  lsyscache.c      - 常用查找函数 (get_opcode, get_func_rettype 等)
```

**系统表定义方式（以 `pg_class.h` 为例）：**
```c
CATALOG(pg_class,1259,RelationRelationId) BKI_BOOTSTRAP BKI_ROWTYPE_OID(83,...)
{
    Oid   oid;
    NameData relname;
    Oid   relnamespace BKI_DEFAULT(pg_catalog) BKI_LOOKUP(pg_namespace);
    // ...
};
```
`CATALOG` 宏展开后生成 `typedef struct FormData_pg_class`。`BKI_BOOTSTRAP` 标记此表在 bootstrap 阶段创建。`BKI_LOOKUP` 表示值需要从另一个系统表的 OID 查找。

**RelCache（`src/backend/utils/cache/relcache.c`）：**
- `RelationIdGetRelation(Oid)` — 用 OID 打开 Relation，返回 `Relation`（即 `RelationData *`）
- `RelationData` 包含：TupleDesc（列描述符）、索引列表、trigger 列表、规则列表、分区键信息、AM 引用等
- Relation 打开时加 `AccessShareLock`，关闭时 `RelationClose()` 释放
- 通过共享 invalidation 消息保持缓存一致性（一个 backend 修改系统表后，commit 时发送 inval，其他 backend 在下一次事务开始时处理）

**共享 Invalidation 消息类型（`src/include/storage/sinval.h`）：**
- Catcache inval：使某个系统表缓存条目失效
- Relcache inval：使某个 relation 的 RelationData 失效
- SMGR inval：物理关系文件变更
- Snapshot inval：快照可能需要更新

---

### 12. Postmaster & Process Architecture

**目录结构：**
```
src/backend/postmaster/
  postmaster.c       - 主守护进程：连接监听、fork backend、管理辅助进程
  bgwriter.c         - Background Writer：定期刷脏页
  checkpointer.c     - Checkpointer：写 checkpoint
  walwriter.c        - WAL Writer：定期 fsync WAL
  autovacuum.c       - Autovacuum Launcher & Workers
  pgarch.c           - Archiver：执行 archive_command
  autostashmerge.c   - 自动合并 stash 分支
```

**进程模型（`postmaster.c` 注释第1-23行）：** Postmaster 是总协调进程。每个客户端连接 fork 一个 backend 进程处理。Postmaster 本身不访问共享内存（避免与其他 backend 一起崩溃）。辅助进程也是 fork 出来的子进程。

**Backend 主循环（`src/backend/tcop/postgres.c`）：**
- `PostgresMain()`: 无限循环读取前端消息
  - `SocketBackend()` 或 `InteractiveBackend()` 读取消息类型
  - Q (Query) → `exec_simple_query(query_string)` 完整的 parse→analyze→rewrite→plan→execute 流水线
  - P (Parse) → extended query protocol
  - B (Bind) → extended query protocol
  - E (Execute) → extended query protocol

**Shared Memory & IPC（`src/backend/storage/ipc/`）：** Postmaster 启动时通过 `shmget()` 分配共享内存，所有 backend 通过 `shmat()` 附加。共享内存包含：Buffer Pool、WAL Buffers、LWLock 数组、Lock 表、PGPROC 数组（每 backend 一个）、CLOG/SL RU 缓冲区。

**并行查询（`src/backend/access/transam/parallel.c`）：** 使用动态共享内存（DSM）。`ParallelContext` 管理 worker 组。`Gather` / `GatherMerge` 执行节点从 worker 收集数据。Worker 通过消息队列通信。

---

### 13. Replication（复制）

**目录结构：**
```
src/backend/replication/
  walsender.c                   - WAL Sender（发送WAL给备库）
  walreceiver.c                 - WAL Receiver（备库接收端）
  walreceiverfuncs.c            - pg_wal_receiver_* SQL函数
  slot.c                        - Replication Slot 管理
  syncrep.c                     - Synchronous Replication
  logical/                      - Logical Decoding
    decode.c, decoding.c, logical.c, reorderbuffer.c, snapbuild.c
  pgoutput/                     - 内置逻辑复制 output plugin
    pgoutput.c
  libpqwalreceiver/             - 基于 libpq 的 walreceiver 模块
```

**物理流复制：**
- WAL Sender：`walsender.c` → `XLogSendPhysical()` 发送WAL记录。支持异步和同步模式
- WAL Receiver：`walreceiver.c` → `WalReceiverMain()` 接收并写入 WAL，然后通知 startup 进程 apply
- Replication Slot (`slot.c`)：跟踪每个 standby 消耗的 WAL 位置，防止需要的 WAL 被删除

**同步复制（`src/backend/replication/syncrep.c`）：** 根据 `synchronous_commit` 和 `synchronous_standby_names` 配置，在事务提交时等待 standby 的确认。

**逻辑解码（`src/backend/replication/logical/`）：**
- `ReorderBuffer`：将 WAL 记录重组为按事务提交顺序排列的逻辑变更
- `SnapBuild`：构建历史快照（用于解码期间读取系统表）
- Output Plugin API：`pg_decode_startup/decode/commit/change` 等回调

**逻辑复制/发布订阅：** 使用 `pgoutput.c` 插件，通过 `pg_publication` (发布端) 和 `pg_subscription` (订阅端) 系统表配置。

---

### 14. Common Utilities（基础工具）

**MemoryContext（`src/include/nodes/memnodes.h`, `src/backend/utils/mmgr/`）：**
- 基于区域的内存管理，不是通用的 malloc/free
- `MemoryContextMethods` 虚函数表：`alloc`, `free_p`, `realloc`, `reset`, `delete_context`
- `AllocSet` 是默认实现：使用 2 的幂次方 freelist + malloc 底层
- `MemoryContextReset(ctx)` 一次性释放上下文中的所有内存（executor 每元组重新使用）
- 关键上下文：`TopMemoryContext` → `CacheMemoryContext` / `MessageContext` → `CurTransactionContext`

**Error Handling（`src/include/utils/elog.h`, `src/backend/utils/error/elog.c`）：**
```c
ereport(ERROR, (errcode(ERRCODE_UNDEFINED_TABLE), errmsg("relation \"%s\" does not exist", relname)));
```
- `ERROR` 级别通过 `siglongjmp` 跳出，回滚当前事务
- `FATAL`：中止连接；`PANIC`：关闭整个服务器
- 错误上下文栈（`error_context_stack`）：每个模块可以压栈/弹出，在错误报告中提供定位信息

**节点系统（`src/include/nodes/nodes.h`, `src/include/nodes/nodetags.h`）：**
- 所有树节点以 `NodeTag type` 开头（tag 在 `nodetags.h` 中枚举定义，由 `gen_node_support.pl` 自动生成）
- `makeNode(Type)`：分配并打标签
- `copyObject(ptr)`：深拷贝节点树
- `equal(a, b)`：结构相等比较
- `nodeToString(ptr)` / `stringToNode(str)`：序列化/反序列化（用于 parallel worker 间传输 plan tree）
- `castNode(Type, ptr)`：Debug 构建中验证节点类型

**List（`src/include/nodes/pg_list.h`）：** 最常用的数据结构。`List *` 是 `void *` 链表。`foreach(lc, list)`, `lfirst()`, `lappend()`, `lcons()`。也支持 `ilist`（集成双向链表）。

**Bitmapset（`src/include/nodes/bitmapset.h`）：** 高效的整数集合。内部用 bitmap 实现。常用于表示属性号集合、关系 ID 集合。

**Hash Tables（`src/include/utils/hsearch.h`）：**
```c
HTAB *hash_create("name", nelem, &ctl, flags);
void *hash_search(htab, keyPtr, action, &found);
// action: HASH_ENTER (插入或返回已存在), HASH_FIND (只查找), HASH_REMOVE
```

**GUC（Grand Unified Configuration, `src/backend/utils/misc/guc_tables.c`）：** 配置参数系统。`DefineCustomBoolVariable()` 等 API 定义参数。支持多种上下文（`PGC_POSTMASTER` 需重启，`PGC_USERSET` 随时可改）。

**Datum（`src/include/postgres.h`）：** `typedef uintptr_t Datum` — 统一的值传递类型。`DatumGetInt32(d)`, `Int32GetDatum(i)` 等宏进行类型转换。支持 pass-by-value (<= 8 bytes on 64-bit) 和 pass-by-reference (pointer stored in Datum)。
