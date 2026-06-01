# Planner/Optimizer — 优化器总览与编排

> 源码目录：`src/backend/optimizer/`（**必读** `src/backend/optimizer/README`）
> 核心入口：`standard_planner()` — `src/backend/optimizer/plan/planner.c:351`
> 上游：[03-rewriter](03-rewriter.md)（Query）　下游：[06-executor-overview](06-executor-overview.md)（PlannedStmt）
> 配套：[05-planner-paths-joins](05-planner-paths-joins.md)（路径、代价、join 枚举细节）

---

## 1. 职责

把逻辑查询树 `Query`（"要什么"）转换为最优物理执行计划 `PlannedStmt`（"怎么做"）。优化器是**基于代价（cost-based）** 的：枚举可行的执行方式，用代价模型估算每种方式的开销，选最便宜的。

优化器要回答的核心问题：
- 每张表用什么方式扫描？（顺序扫描 / 索引扫描 / 位图扫描）
- 多表按什么**顺序**连接？（join order，搜索空间随表数阶乘增长）
- 每个 join 用什么**算法**？（嵌套循环 / 归并 / 哈希）
- 聚合、排序、去重放在哪一步、用什么实现？
- 能否并行？

---

## 2. 目录结构与分工

`README` 开篇即点明四个子目录的分工：

```
src/backend/optimizer/
  plan/    生成最终输出 Plan
    planner.c     planner()/standard_planner() 顶层；subquery_planner/grouping_planner
    planmain.c    query_planner()：构建基表、驱动 join 搜索
    createplan.c  Path 树 → Plan 树的转换
    initsplan.c   初始化 join 搜索：分发 qual 到 RestrictInfo/joininfo
    setrefs.c     Plan 完成后修正 Var 引用、扁平化 rtable
    subselect.c   子查询/子计划处理（SubPlan、InitPlan）
    analyzejoins.c  无用 join 消除、唯一性分析
  path/    枚举所有可行的扫描/连接方式（生成 Path）
    allpaths.c    为每个基表建所有扫描 Path；驱动 join 层级
    indxpath.c    索引扫描 Path 生成
    joinpath.c    为一对 rel 生成 nestloop/merge/hash 三类 join Path
    joinrels.c    join 顺序枚举（动态规划）
    costsize.c    代价估算函数
    equivclass.c  等价类（= 的传递闭包），推导隐含 join/过滤条件
    pathkeys.c    PathKey（排序序）管理
    tidpath.c     TID 扫描
  prep/    特殊情况预处理
    prepjointree.c  子查询提升、JOIN 扁平化、外连接简化
    prepqual.c      WHERE 规范化（转 CNF/简化布尔）
    preptlist.c     target list 预处理
    prepagg.c       聚合预处理（拆分 aggref）
  util/    工具
    relnode.c     RelOptInfo 的建立与查找
    restrictinfo.c  RestrictInfo（带缓存元数据的 qual）
    plancat.c     从系统表/relcache 取统计信息（行数、页数、索引、直方图）
    clausesel.c   选择率（selectivity）估算
    pathnode.c    Path 节点构造与 add_path（路径剪枝）
  geqo/    遗传算法优化器（表数超过 geqo_threshold 时的近似搜索）
```

---

## 3. 核心数据结构

### 3.1 PlannerGlobal — 跨子查询的全局状态

`standard_planner()` 第一件事就是建 `PlannerGlobal`（`planner.c:371`），它在整条命令的所有子查询层级间共享（`planner.c:365-370` 注释）。保存：所有 subplan、最终扁平化的 rtable（`finalrtable`）、结果关系、依赖的 relation OID（用于计划失效）、param 类型等。

### 3.2 PlannerInfo（root）— 每个查询层级一份

`src/include/nodes/pathnodes.h` 的 `PlannerInfo`（习惯命名为 `root`）。每个 `Query`（含每个子查询）对应一个 `PlannerInfo`，是优化某一层查询时无处不在的上下文：

- `parse`：正在优化的 `Query`。
- `simple_rel_array` / `simple_rte_array`：按 RT 下标索引的 `RelOptInfo` 与 `RangeTblEntry` 数组。
- `join_rel_list` / `join_rel_hash`：已建立的 join `RelOptInfo`。
- `eq_classes`：等价类列表。
- `canon_pathkeys`：规范化的 PathKey。
- `glob`：指回共享的 `PlannerGlobal`。

### 3.3 RelOptInfo —— 一个关系（基表或 join 结果）

`README` 的核心抽象：为查询中**每个基表**和**每个被考虑的 join 组合**建一个 `RelOptInfo`。关键不变量：**给定一组基表，只有一个 join RelOptInfo**——`{A,B,C}` 无论先 join 谁，都是同一个 RelOptInfo，不同的构建方式表现为它 `pathlist` 中不同的 Path。

```c
typedef struct RelOptInfo {
    Relids      relids;             // 本 rel 覆盖的基表 RT 下标集合（Bitmapset）
    double      rows;               // 估算输出行数
    List       *pathlist;           // 可行 Path 列表
    Path       *cheapest_total_path;
    Path       *cheapest_startup_path;
    List       *indexlist;          // 可用索引（IndexOptInfo）
    List       *baserestrictinfo;   // 只涉及本 rel 的过滤条件
    List       *joininfo;           // 涉及本 rel 的 join 条件
    ...
} RelOptInfo;
```

### 3.4 Path —— 一种执行方式

`README`：Path 与 Plan 近乎一一对应，但 Path 省略执行期才需要的信息、保留规划期需要的信息（如 `pathkeys`、参数化信息）。Path 是树：顶层节点是最后施加的操作（如最后一个 join），其 `outerpath`/`innerpath` 子树是输入。

```c
typedef struct Path {
    NodeTag     type;
    NodeTag     pathtype;       // 将变成哪种 Plan 节点
    RelOptInfo *parent;         // 这个 Path 产出哪个 rel
    Cost        startup_cost;   // 拿到第一行的代价
    Cost        total_cost;     // 拿到所有行的代价
    List       *pathkeys;       // 输出的排序序
    double      rows;
    ...
} Path;
```

子类型：`IndexPath`、`BitmapHeapPath`、`NestPath`、`MergePath`、`HashPath`、`AppendPath`、`AggPath`、`SortPath` 等。详见 [05-planner-paths-joins](05-planner-paths-joins.md)。

---

## 4. 核心算法：standard_planner 的编排

`standard_planner()`（`planner.c:351`）的主干：

```
1. 建 PlannerGlobal（glob）                       planner.c:371
2. 评估能否并行（max_parallel_hazard 等）          planner.c:397
   - standalone / 有数据修改 / cursor / parallel-unsafe 函数 → 禁并行
3. 计算 tuple_fraction（LIMIT/cursor 影响"只取前 N 行"的偏好）
4. subquery_planner(glob, parse, ...)              planner.c:775
   ── 递归优化单层查询，返回 final RelOptInfo
5. final_rel = fetch_upper_rel(UPPERREL_FINAL)
   best_path  = final_rel->cheapest_total_path
6. top_plan = create_plan(root, best_path)         path 树 → plan 树
7. set_plan_references(...)                          扁平化 rtable、修正 Var
8. 组装 PlannedStmt（含 subplans、rtable、依赖 OID 等）
```

### 4.1 subquery_planner —— 单层查询优化

`subquery_planner()`（`planner.c:775`）对一层 `Query` 做预处理 + 主优化：

1. **子链接/子查询提升**：`pull_up_sublinks()` 把 `EXISTS`/`IN` 子查询尽量转成半连接（semijoin）；`pull_up_subqueries()` 把简单子查询/视图内联到上层（消除子查询边界，扩大 join 搜索空间）。
2. **表达式预处理**：`preprocess_expression()` 常量折叠（`eval_const_expressions`）、简化布尔、内联简单 SQL 函数。
3. **JOIN 树整理**：`prepjointree.c` 扁平化显式 JOIN（受 `join_collapse_limit`/`from_collapse_limit` 约束）、化简外连接。
4. **qual 分发**：`distribute_qual_to_rels()`（`initsplan.c`）把 WHERE/ON 条件包成 `RestrictInfo`，分发到相关 rel 的 `baserestrictinfo` 或 `joininfo`，并建立等价类。
5. **主优化**：`grouping_planner()`（`planner.c:1775`）→ `query_planner()`（`planmain.c`）做扫描/join 路径搜索；之后在"上层关系"（upper rels）上叠加 grouping、window、distinct、ordering、limit 的 Path。

### 4.2 grouping_planner 的"上层关系"流水线

`query_planner()` 产出连接好所有基表的 `final scan/join rel` 后，`grouping_planner()` 依次构造一串 **upper relation**（`UPPERREL_*`），每一层在前一层的 cheapest path 之上叠加一种操作并比较代价：

```
scan/join rel
  → UPPERREL_GROUP_AGG    （GROUP BY / 聚合：HashAgg vs GroupAgg）
  → UPPERREL_WINDOW       （窗口函数）
  → UPPERREL_DISTINCT     （DISTINCT）
  → UPPERREL_ORDERED      （ORDER BY）
  → UPPERREL_FINAL        （LockRows / LIMIT / ModifyTable）
```

每层都可能有多个 Path（如聚合可选 hash 或 sort 两条路），靠 `add_path()` 的代价剪枝保留有用的。

### 4.3 Path → Plan：createplan

`create_plan()`（`createplan.c`）自顶向下递归把最优 Path 树翻译成 Plan 树：`SeqScanPath→SeqScan`、`HashPath→HashJoin+Hash`、`AggPath→Agg` 等。随后 `set_plan_references()`（`setrefs.c`）把每个 Plan 节点里的 `Var` 从"基于 rtable 下标"改写为"基于子节点输出列下标"，并把所有子查询 rtable 合并进 `glob->finalrtable`。

---

## 5. 设计模式

- **生成-评估-择优（Generate & Cost-based selection）**：枚举 Path → 代价估算 → `add_path` 剪枝 → 取 cheapest。整个优化器是这一模式的层层嵌套。
- **不变式驱动的去重（一个基表集合一个 RelOptInfo）**：把"join 顺序"与"join 结果"解耦，使动态规划成为可能。
- **Path/Plan 分离**：规划期用轻量 Path（带 cost、pathkeys、参数化），定稿后才膨胀为 Plan，避免在搜索过程中携带执行期负担。
- **两层上下文（Global / PlannerInfo）**：跨子查询的共享放 `PlannerGlobal`，每层私有放 `PlannerInfo`，与 [02-analyzer](02-analyzer.md) 的 ParseState 异曲同工。
- **等价类做谓词推导**：把 `=` 当作等价关系求传递闭包，自动派生隐含条件（`a=b ∧ b=c ⇒ a=c`），扩大可用 join/过滤条件。
- **GEQO 兜底**：当精确动态规划的搜索空间爆炸（表数 > `geqo_threshold`，默认 12），切换到遗传算法做近似搜索，但每个候选 join 仍交给 `/path` 生成真实 Path。

---

## 6. 架构编排

```
pg_plan_queries()                            tcop/postgres.c:987
  └─ planner() → standard_planner()           optimizer/plan/planner.c:351
       ├─ PlannerGlobal 初始化
       ├─ 并行可行性评估
       └─ subquery_planner()                  planner.c:775
            ├─ pull_up_sublinks / pull_up_subqueries   prep/prepjointree.c
            ├─ preprocess_expression（常量折叠等）       plan/planner.c
            ├─ distribute_qual_to_rels（建 RestrictInfo/等价类）  plan/initsplan.c
            └─ grouping_planner()              planner.c:1775
                 └─ query_planner()            plan/planmain.c
                      ├─ set_base_rel_pathlists  → 见 05：扫描 Path
                      └─ make_rel_from_joinlist  → 见 05：join 枚举
                 → upper rels（agg/window/distinct/order/final）
       ├─ create_plan(best_path)              plan/createplan.c
       └─ set_plan_references                 plan/setrefs.c
       → PlannedStmt
```

---

## 7. 动手探索

```sql
-- 看优化器最终选择
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;

-- 强行关闭某种方法，逼优化器换路，理解代价比较
SET enable_hashjoin = off;  SET enable_seqscan = off;

-- 观察 join 搜索的两种模式
SET geqo = off;  SET join_collapse_limit = 1;   -- 固定 join 顺序为书写顺序
SET from_collapse_limit = 8;

-- 看优化器掌握的统计
SELECT relname, reltuples, relpages FROM pg_class WHERE relname='t';
SELECT * FROM pg_stats WHERE tablename='t';
```

调试：`SET debug_print_plan = on; SET client_min_messages = debug1;` 可在日志里打印完整 Plan 树。在 `standard_planner` 出口 `call pprint(result)`。

---

## 相关模块

- 细节续篇：[05-planner-paths-joins](05-planner-paths-joins.md)
- 下游：[06-executor-overview](06-executor-overview.md)
- 统计来源：[../catalog/40-catalog](../catalog/40-catalog.md)（pg_statistic）
- 计划缓存：plancache（generic vs custom plan）
