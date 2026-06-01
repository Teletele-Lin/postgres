# Planner — 路径、代价估算与 Join 枚举

> 源码目录：`src/backend/optimizer/path/`、`src/backend/optimizer/util/`
> 配套总览：[04-planner-overview](04-planner-overview.md)

本文承接总览，深入优化器的"引擎室"：如何为关系生成 Path、如何估代价、如何用动态规划枚举 join 顺序。

---

## 1. 职责

- 为每个基表生成所有有用的**扫描路径**（顺序/索引/位图/TID）。
- 为每对关系生成所有可行的**连接路径**（nestloop/merge/hash），并按动态规划逐层组合。
- 用代价模型给每条 Path 估 `startup_cost` 和 `total_cost`，并用 `add_path()` 剪掉劣势路径。
- 处理排序序（PathKey）、参数化路径（parameterized path）、等价类推导。

---

## 2. 扫描路径的生成

`set_base_rel_pathlists()` → `set_rel_pathlist()`（`allpaths.c`）为每个基表生成扫描路径：

| 路径类型 | 生成函数 | 适用条件 |
|---------|---------|---------|
| `SeqScan` | `add_path(create_seqscanpath(...))` | 总是可行 |
| `IndexScan` / `IndexOnlyScan` | `create_index_paths()`（`indxpath.c`） | 有可用索引；index-only 要求 VM all-visible |
| `BitmapHeapScan` | `create_bitmap_heap_path()` | 多索引/低选择率，先攒 TID 位图再回表 |
| `TidScan` | `tidpath.c` | WHERE 含 `ctid = ...` |
| 并行 `SeqScan` 等 | partial path | 表足够大且允许并行 |

### 2.1 索引路径匹配

`indxpath.c` 的核心是把 `baserestrictinfo`/`joininfo` 中的子句与索引列匹配，判断哪些子句能作为 **index condition**（下推到索引层）而非 **filter**（回表后过滤）。还会考虑索引隐含的排序序（生成 `pathkeys`），让 ORDER BY 可以"免费"从索引顺序获得。

### 2.2 参数化路径（parameterized path）

对 nestloop 内表，某些扫描可以接收外表当前行的值作为参数（如 `inner.id = outer.id` 用 inner 上的索引）。这种 Path 带 `param_info`，记录它依赖哪些外部 rel。它的代价是"每次重扫"的代价，组合进 nestloop 时按外表行数放大。

---

## 3. 代价估算（costsize.c）

### 3.1 代价参数（GUC）

代价是抽象单位，以"顺序读一个页"为基准 1.0：

| 参数 | 默认值 | 含义 |
|------|-------|------|
| `seq_page_cost` | 1.0 | 顺序读一页 |
| `random_page_cost` | 4.0 | 随机读一页（索引扫描更贵） |
| `cpu_tuple_cost` | 0.01 | 处理一行的 CPU 代价 |
| `cpu_index_tuple_cost` | 0.005 | 处理一条索引项 |
| `cpu_operator_cost` | 0.0025 | 执行一个运算符/函数 |
| `parallel_setup_cost` / `parallel_tuple_cost` | 1000 / 0.1 | 并行启动与跨进程传一行 |

### 3.2 关键代价函数

`cost_seqscan()`（`costsize.c:270`）：

```
startup_cost = 0
run_cost     = seq_page_cost * relpages          // 读所有页
             + cpu_tuple_cost * reltuples         // 处理所有行
             + cpu_operator_cost * qual代价 * reltuples  // 过滤每行
total_cost   = startup_cost + run_cost
```

`cost_index()`（`costsize.c:545`）：索引树遍历代价 + 叶子页扫描 + **回表代价**。回表用 `random_page_cost`，但用 `index_pages_fetched()`（Mackert-Lohman 公式）估算"考虑缓存命中后实际要读多少页"，避免对大表索引高估。

`final_cost_hashjoin()`（`costsize.c:4416`）：内表扫描 + 建哈希表（每行 `cpu_operator_cost` 算 hash）+ 外表扫描 + 探测；若内表超 `work_mem` 需分批（batch），追加溢出读写代价。

nestloop：`outer_path总代价 + outer行数 × inner每次重扫代价`，所以参数化的便宜内表扫描对 nestloop 极重要。

mergejoin：两侧排序代价（若输入未排好）+ 一次同步归并扫描；代价对"已有合适 pathkeys"的输入特别友好。

### 3.3 选择率（selectivity）

行数估算依赖选择率：一个谓词过滤后剩多少比例的行。`clausesel.c` + 各类型的 `*sel` 函数（如 `eqsel`、`scalarltsel`）查 `pg_statistic`（直方图、MCV、n_distinct、相关性）计算。join 选择率由 `eqjoinsel` 等估算。统计来自 `ANALYZE`，由 `plancat.c` 在规划时从 relcache/syscache 读出。

---

## 4. Join 枚举：动态规划

### 4.1 顶层入口

`make_rel_from_joinlist()`（`allpaths.c:3843`）拿到 join 列表后，按表数量决定走精确还是近似搜索：表数 > `geqo_threshold`（默认 12）走 GEQO（`geqo_eval`），否则走 `standard_join_search()`（`allpaths.c:3948`）。两者都可被钩子 `join_search_hook` 替换（`pg_hint_plan` 等扩展用它）。

### 4.2 standard_join_search 的动态规划

`allpaths.c:3948`，注释（`:3960-3968`）就是算法本身：

```c
/*
 * 简单的"动态规划"算法：先求所有两项 join，再从两项 join 与单项求三项
 * join，再四项……直到把所有项 join 成一个 rel。
 * root->join_rel_level[j] 是所有 j 项 rel 的列表。
 */
root->join_rel_level[1] = initial_rels;          // 第 1 层 = 各基表
for (lev = 2; lev <= levels_needed; lev++)
{
    join_search_one_level(root, lev);            // 构造本层所有 j 项 join rel
    foreach(rel in root->join_rel_level[lev])
    {
        generate_partitionwise_join_paths(...);  // 分区智能连接
        generate_useful_gather_paths(...);       // 并行收集
        set_cheapest(rel);                        // 选出本 rel 最优 path
    }
}
```

**层级含义**：`join_rel_level[j]` = 恰好包含 j 个 jointree 项的所有 join rel。第 lev 层从更低层组合而来（如 4 项 = 1项×3项 = 2项×2项），靠 `make_join_rel()` 去重到同一个 RelOptInfo。

### 4.3 join_search_one_level

`join_search_one_level()`（`joinrels.c:78`）在第 lev 层穷举可连接的 rel 对：

- **有 join clause 的对优先**：尽量只考虑有可用连接条件的组合（避免笛卡尔积）。
- **bushy / left-deep / right-deep**：通过组合 `(level-1, 1)`、`(level-2, 2)` 等划分，允许 bushy plan（两个多表 rel 互相 join），不限于左深树。
- **合法性检查**：外连接、`IN/EXISTS` 引入的 join 顺序约束由 `join_is_legal()` 把关；无连接条件时才退化为允许笛卡尔积。

### 4.4 make_join_rel 与三种 join path

`make_join_rel(root, rel1, rel2)`（`joinrels.c:699`）：

1. 查/建代表 `rel1 ∪ rel2` 的 join `RelOptInfo`（用 `relids` 做 key 去重）。
2. 调 `add_paths_to_joinrel()` → `joinpath.c`，为这对输入尝试：
   - `sort_inner_and_outer` → MergeJoin（必要时加 Sort）
   - `match_unsorted_outer` → NestLoop + 利用外表已有顺序的 MergeJoin
   - `hash_inner_and_outer` → HashJoin
   每种可行方式生成一个 Path，`add_path()` 按代价剪枝。

### 4.5 add_path —— 路径剪枝

`add_path()`（`pathnode.c:459`）维护 `RelOptInfo.pathlist` 的"帕累托前沿"：新 Path 若在 **(total_cost, startup_cost, pathkeys 排序性, 参数化, 并行安全)** 多维上被某个已有 Path 全面支配，则丢弃；反之若它支配了旧 Path，则删旧留新。所以 pathlist 里留下的都是某种意义上"不可被替代"的路径（比如最快总时间的、最快出首行的、能提供某种排序序的）。

---

## 5. 等价类与 PathKey

### 5.1 等价类（equivalence class，equivclass.c）

把所有 `=` 子句视作等价关系求传递闭包。`a.x = b.x` 和 `b.x = c.x` 生成一个含 `{a.x, b.x, c.x}` 的等价类。好处：
- 自动派生隐含 join 条件 `a.x = c.x`，多一条可用连接路径。
- 把 `a.x = 常量` 传播给类中所有成员，多出过滤条件下推机会。
- 排序/分组键可在等价成员间自由替换。

### 5.2 PathKey（pathkeys.c）

`PathKey` 描述"按某等价类、某排序算子、某方向、某 NULLS 位置排序"。它统一表达：索引扫描产生的顺序、Sort 节点产生的顺序、mergejoin 要求的顺序、ORDER BY/GROUP BY 需要的顺序。优化器靠比较 pathkeys 判断"某 Path 的输出顺序是否满足上层需求"，从而决定能否省掉 Sort。

---

## 6. 设计模式

- **动态规划 + 子结构去重**：`join_rel_level[]` 是经典 DP 表；"一个 relids 一个 RelOptInfo"保证子问题只解一次。
- **帕累托剪枝（add_path）**：用多维支配关系而非单一代价裁剪搜索空间，兼顾 total/startup/排序/参数化。
- **代价模型即可插拔参数**：所有代价由 GUC 参数线性组合，DBA 可按硬件调（SSD 调低 `random_page_cost`）。
- **等价类做约束传播**：把"相等"提升为一等公民，统一支撑谓词下推、隐含 join、排序替换。
- **钩子（join_search_hook / set_rel_pathlist_hook）**：扩展（如 `pg_hint_plan`）无需改核心即可干预路径与 join 搜索。

---

## 7. 架构编排

```
query_planner()                              plan/planmain.c
  ├─ set_base_rel_sizes / set_base_rel_pathlists   path/allpaths.c
  │     └─ 每个基表：seqscan + create_index_paths + bitmap + tid + 并行
  │           └─ add_path 剪枝 → set_cheapest
  └─ make_rel_from_joinlist                    path/allpaths.c:3843
        ├─ (表多) geqo_eval                     geqo/
        └─ standard_join_search                path/allpaths.c:3948
              for lev in 2..N:
                join_search_one_level           path/joinrels.c:78
                  └─ make_join_rel               joinrels.c:699
                       └─ add_paths_to_joinrel   path/joinpath.c
                            ├─ sort_inner_and_outer  → MergePath
                            ├─ match_unsorted_outer  → NestPath/MergePath
                            └─ hash_inner_and_outer  → HashPath
                set_cheapest(每个 joinrel)
  → final scan/join rel → grouping_planner 叠加上层操作（见 04）
```

---

## 8. 动手探索

```sql
-- 让代价可见
EXPLAIN SELECT * FROM a JOIN b ON a.id=b.id JOIN c ON b.k=c.k;
-- cost=startup..total rows=.. width=.. 每个节点都有

-- 调代价参数观察路径切换
SET random_page_cost = 1.1;   -- 模拟 SSD，更倾向索引扫描
SET work_mem = '4MB';         -- 太小会逼 hashjoin 分批、sort 落盘

-- 关闭某方法，看 add_path 剪枝后的次优路径
SET enable_indexscan = off;

-- 观察等价类推导出的隐含条件
EXPLAIN VERBOSE SELECT * FROM a,b,c WHERE a.x=b.x AND b.x=c.x;
-- 计划里可能出现 a.x=c.x 这条没写的条件
```

---

## 相关模块

- 总览：[04-planner-overview](04-planner-overview.md)
- 执行落地：[06-executor-overview](06-executor-overview.md)、[08-executor-nodes](08-executor-nodes.md)
- 统计数据：[../catalog/40-catalog](../catalog/40-catalog.md)
- 索引能力：[../storage/12-index-am](../storage/12-index-am.md)
