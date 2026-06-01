# Executor — 执行节点详解（扫描 / 连接 / 聚合 / 排序）

> 源码：`src/backend/executor/node*.c`
> 配套：[06-executor-overview](06-executor-overview.md)（火山框架）、[07-executor-expr](07-executor-expr.md)（表达式）

每个 Plan 节点类型对应一个 `nodeXxx.c`，实现 `ExecInitXxx / ExecXxx / ExecEndXxx` 三个函数，并在 `execProcnode.c` 的三个分发 switch 中登记。本文挑选最具代表性的几类深入。

---

## 1. 扫描节点（Scan）

所有扫描节点的 PlanState 继承自 `ScanState`（含 `ss_currentRelation`、`ss_currentScanDesc`、`ss_ScanTupleSlot`），共用 `execScan.c` 的 `ExecScan()` 框架：循环"取一行 → 过 qual → 投影"。

| 节点 | 文件 | 行为 |
|------|------|------|
| SeqScan | `nodeSeqscan.c` | 经 Table AM `table_scan_getnextslot` 顺序读全表；用 `ReadStream` 预读 |
| IndexScan | `nodeIndexscan.c` | `index_getnext_tid` 取 TID → `table_index_fetch_tuple` 回表 |
| IndexOnlyScan | `nodeIndexonlyscan.c` | 仅读索引；靠 VM 判断 all-visible，免回表 |
| BitmapHeapScan | `nodeBitmapHeapscan.c` | 先由 BitmapIndexScan 攒 TID 位图，再按页排序批量回表 |
| TidScan / TidRangeScan | `nodeTidscan.c` 等 | 直接按 `ctid` 取行 |
| FunctionScan / ValuesScan / CteScan | 各对应文件 | FROM 中的函数/VALUES/CTE |
| ForeignScan / CustomScan | `nodeForeignscan.c` 等 | FDW / 扩展自定义扫描 |

`ExecScan()`（`execScan.c`）的统一循环：调 `accessMtd` 拿一行存入 `ss_ScanTupleSlot` → 用 `ExecQual` 过滤 → `ExecProject` 投影输出。这层抽象让所有扫描节点共享过滤/投影逻辑，各自只实现"如何取下一行"。

SeqScan 在 PG 中通过 **ReadStream**（`storage/16`）发起预读，把"决定读哪个块"与"实际异步 I/O"解耦，提升顺序扫描吞吐。

---

## 2. 连接节点（Join）

三种 join 算法都继承 `JoinState`（含 `jointype`、`joinqual`、`lefttree`=outer、`righttree`=inner）。

### 2.1 NestLoop（`nodeNestloop.c`）

最朴素：对外表每一行，重扫一遍内表找匹配。

```
foreach outer_row:
    ExecReScan(inner)            // 内表带参数化时，用 outer 当前值重扫
    foreach inner_row:
        if joinqual(outer,inner): emit
```

关键在与**参数化内表**配合：当内表是参数化索引扫描（`inner.id = outer.id` 走索引），每次 rescan 只取匹配行，nestloop 退化为高效的索引嵌套循环。`Memoize` 节点（`nodeMemoize.c`）可缓存内表重扫结果，避免相同外部参数重复扫描。

### 2.2 MergeJoin（`nodeMergejoin.c`）

要求两侧按连接键有序（输入若无序则上面加 Sort）。两个游标同步前移做归并：

```
while outer 与 inner 都未尽:
    比较 outer键 与 inner键
      <  : 推进 outer
      >  : 推进 inner
      == : 输出所有键相等的 outer×inner 组合（处理重复键的"标记/回退"）
```

对已排序输入（如两侧都走索引顺序）几乎线性、内存占用小，且天然产出有序结果。

### 2.3 HashJoin（`nodeHashjoin.c`）—— Hybrid Hash Join

基于 **hybrid hash join** 算法（`nodeHashjoin.c:13-24` 引 Zeller & Gray 1990）。核心：扫内表建哈希表，再扫外表探测。内表放不下 `work_mem` 时分**批（batch）** 溢出到临时文件。

**批数始终是 2 的幂**（`nodeHashjoin.c:35-36`），不够时翻倍。串行与并行的策略不同（`:38-48`）：
- **串行**：惰性测量——装载某批时才判断是否超 `work_mem`，超了就 dump 哈希表、把元组重新分配到其它批文件或当前批。
- **并行**：在 build 阶段一次性完成所有批数变更；增批时把所有元组重分配到全新批文件，再检查每批是否符合空间预算。

二者都"尽力而为"（`:50-55`）：某批装不下就翻倍批数；若翻倍后某批保留了全部或零个元组（说明翻倍无效，多半是数据倾斜），则**全局禁用增批**，后续超限的批直接硬撑。

**状态机**（`nodeHashjoin.c:182-188`）—— `ExecHashJoinImpl()` 是个状态机：

```c
#define HJ_BUILD_HASHTABLE          1   // 扫内表建哈希表（首次进入）
#define HJ_NEED_NEW_OUTER           2   // 取下一条外表元组，定位其桶
#define HJ_SCAN_BUCKET              3   // 扫该桶，输出匹配
#define HJ_FILL_OUTER_TUPLE         4   // 左/全外连接：补未匹配外表行
#define HJ_FILL_INNER_TUPLES        5   // 右/全外连接：补未匹配内表行
#define HJ_FILL_OUTER_NULL_TUPLES   6
#define HJ_FILL_INNER_NULL_TUPLES   7
```

`HJ_FILL_OUTER`/`HJ_FILL_INNER` 宏（`:192-194`）按是否存在 NULL 端 slot 判断当前 join 类型是否需要补空行，从而支持 INNER/LEFT/RIGHT/FULL/SEMI/ANTI 全部语义。

**并行 HashJoin**：通过共享内存哈希表 + `Barrier` IPC 同步各 worker 的 build/probe 阶段，让多个 worker 协作构建同一张哈希表（见 [../process/53-parallel-query](../process/53-parallel-query.md)）。

---

## 3. 聚合（`nodeAgg.c`）

支持四种策略（`AggStrategy`）：`AGG_PLAIN`（无 GROUP BY 的整表聚合）、`AGG_SORTED`（输入按 GROUP 键有序，相邻分组）、`AGG_HASHED`（哈希分组）、`AGG_MIXED`（部分键 hash、部分 sort，用于 GROUPING SETS）。

### 3.1 聚合执行三步曲

每个聚合函数由系统表 `pg_aggregate` 定义 `transfn`（状态转移）、`finalfn`（最终化）、`initcond`（初值）：

```
state = initcond
foreach input_row in group:
    state = transfn(state, row)      // 如 sum 的 int4_sum
result = finalfn(state)              // 如 avg 的 numeric_avg（sum/count）
```

`AGG_HASHED` 为每个分组在哈希表里存一份 `state`；`AGG_SORTED` 只需为"当前分组"存 `state`，分组边界一变就 finalize 并输出，内存恒定。

### 3.2 partial aggregation（并行聚合）

`AggSplit` 把聚合拆成两段以支持并行：

```
各 worker: Partial Agg  → 产出中间 state（用 serialfn 序列化）
           ↓ Gather
leader:    Final Agg     → combinefn 合并各 worker 的 state → finalfn
```

`combinefn`（合并两个 state）、`serialfn`/`deserialfn`（state 的跨进程传输）也在 `pg_aggregate` 中定义。这让 `sum/count/avg` 等能并行：每个 worker 聚合自己那份数据，leader 合并部分结果。

---

## 4. 排序与物化

| 节点 | 文件 | 说明 |
|------|------|------|
| Sort | `nodeSort.c` | 包装 `tuplesort.c`：内存放得下用快排，放不下走外部归并排序（多路归并临时文件） |
| IncrementalSort | `nodeIncrementalSort.c` | 输入已按前缀键有序时，仅在小批内排剩余键，省内存并能早出行 |
| Material | `nodeMaterial.c` | 把子节点输出缓存到 tuplestore，供上层多次重扫（如 mergejoin 内表回退） |
| Unique | `nodeUnique.c` | 对已排序输入去相邻重复（DISTINCT 的一种实现） |
| Limit | `nodeLimit.c` | OFFSET/LIMIT 早停，是火山模型早停的典型受益者 |
| Hash | `nodeHash.c` | 为 HashJoin 构建哈希表的"内表侧" |

`tuplesort.c`（`src/backend/utils/sort/`）是排序核心：用 `work_mem` 决定内存/外部排序，外部排序生成有序 run 再多路归并，落盘用 MinimalTuple 紧凑格式。

---

## 5. DML 执行（`nodeModifyTable.c`）

INSERT/UPDATE/DELETE/MERGE 统一由 `ExecModifyTable()` 驱动，是个"消费子计划输出并施加修改"的节点：

- 子计划（扫描/连接）产出待修改行（UPDATE/DELETE 带 `ctid` junk 列定位目标）。
- 调 Table AM 的 `tuple_insert/update/delete`（见 [../storage/10-table-am](../storage/10-table-am.md)）。
- 维护索引（`execIndexing.c`）、外键与 CHECK 约束、行/语句级触发器（BEFORE/AFTER）、分区路由（`execPartition.c`）、`RETURNING`、`ON CONFLICT`。
- **EvalPlanQual**：READ COMMITTED 下 UPDATE/DELETE 遇到并发修改的行，重新读最新版本并对其重跑计划的 qual，决定是否仍要修改（见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)）。

**MERGE**（`README:231-247`）：规划期 `transform_MERGE_to_join` 把目标表与源关系构造成 join（含 NOT MATCHED 时为外连接），ModifyTable 从 join 取行；若目标侧行标识列为 NULL 说明该源行未匹配，按 WHEN NOT MATCHED 动作处理，否则按 WHEN MATCHED 处理。

---

## 6. 并行收集与 Append

- **Gather / GatherMerge**（`nodeGather.c` / `nodeGatherMerge.c`）：并行计划的"汇合点"。leader 启动 worker，各 worker 跑同一段 partial plan，Gather 从共享 tuple queue 收集结果。GatherMerge 在收集时按序归并（保持 ORDER BY）。
- **Append / MergeAppend**（`nodeAppend.c` / `nodeMergeAppend.c`）：UNION ALL 与分区表的多子计划拼接。MergeAppend 保持各子计划有序输入的整体有序。Append 支持运行期分区裁剪（run-time partition pruning）与并行 Append。

---

## 7. 设计模式

- **三相分发模板（Init/Proc/End）**：每个节点实现统一接口，`execProcnode.c` 三个 switch 集中分发；加新节点只需在三处登记（`execProcnode.c:21-23` 注释说明合并到一个文件正是为了便于同步）。
- **状态机驱动复杂算子**：HashJoin 用显式 `HJ_*` 状态机统一表达 build/probe/外连接补行/分批，把"可被反复调用拉取一行"的迭代器语义与多阶段算法调和。
- **算法策略可选（AggStrategy / Sort 内外）**：同一逻辑算子按数据规模与输入顺序选不同物理实现。
- **拆分以并行（AggSplit / partial path + Gather）**：把算子拆成 partial + final 两段，配合 combinefn/serialfn 实现可并行聚合。
- **共享框架抽公共逻辑（ExecScan）**：扫描类节点共享"取行→过滤→投影"，各自只填"如何取行"。

---

## 8. 架构编排（一个 HashJoin 计划的运行）

```
ExecInitNode(HashJoin)
  ├─ ExecInitNode(outer 子计划: SeqScan a)
  └─ ExecInitNode(inner: Hash → SeqScan b)
ExecProcNode(HashJoin) 反复调用：
  HJ_BUILD_HASHTABLE: 拉空 inner(Hash) 建哈希表（可能分批落盘）
  HJ_NEED_NEW_OUTER : ExecProcNode(outer) 取一行，算 hash 定位批/桶
  HJ_SCAN_BUCKET    : 扫桶，joinqual 通过则 ExecProject 输出
  ...（外连接时进入 HJ_FILL_* 补空行）
ExecEndNode(HashJoin): 释放哈希表、临时文件、子节点
```

---

## 9. 动手探索

```sql
-- 看不同 join 算法与分批
EXPLAIN ANALYZE SELECT * FROM a JOIN b ON a.id=b.id;   -- Hash Join: Batches: N
SET work_mem='64kB';                                   -- 逼 HashJoin 多批、Sort 落盘
SET enable_hashjoin=off;                               -- 改走 MergeJoin/NestLoop

-- 聚合策略切换
EXPLAIN ANALYZE SELECT k, count(*) FROM t GROUP BY k;  -- HashAggregate vs GroupAggregate
EXPLAIN ANALYZE SELECT count(*) FROM big;              -- 大表可能出现 Partial Aggregate + Gather

-- 增量排序
EXPLAIN ANALYZE SELECT * FROM t ORDER BY a, b;         -- a 上有索引时可能 Incremental Sort
```

---

## 相关模块

- 火山框架与 slot：[06-executor-overview](06-executor-overview.md)
- 表达式/qual：[07-executor-expr](07-executor-expr.md)
- 数据存取：[../storage/10-table-am](../storage/10-table-am.md)、[../storage/13-nbtree](../storage/13-nbtree.md)
- 并行：[../process/53-parallel-query](../process/53-parallel-query.md)
