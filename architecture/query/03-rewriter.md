# Rewriter — 查询重写器

> 源码目录：`src/backend/rewrite/`
> 核心入口：`QueryRewrite()` — `src/backend/rewrite/rewriteHandler.c:4781`
> 上游：[02-analyzer](02-analyzer.md)（Query）　下游：[04-planner-overview](04-planner-overview.md)（Query 列表）

---

## 1. 职责

在分析与规划之间，对 `Query` 施加 **规则系统（rule system）** 变换，把"逻辑上引用视图/受规则约束的查询"改写成"直接操作基表的查询"。具体包括：

- **视图展开**：`SELECT ... FROM a_view` → 用视图定义的子查询替换该 RTE。
- **规则（RULE）应用**：`CREATE RULE` 定义的 ON SELECT/INSERT/UPDATE/DELETE DO [INSTEAD] 规则。
- **行级安全（RLS）**：把 `pg_policy` 中的策略表达式注入 WHERE/CHECK。
- **可更新视图**：对 `INSERT/UPDATE/DELETE` 到简单视图的语句，改写为对基表的操作。

一个输入 `Query` 可能产出 **0 个、1 个或多个** `Query`（因为 DO ALSO 规则会追加语句，DO INSTEAD 会替换）。因此入口签名是 `List *QueryRewrite(Query *)`。

历史术语（`rewriteHandler.c:13-17` 注释）：**"retrieve"** 是 PostQUEL 时代的 SELECT；**"RIR"** = Retrieve-Instead-Retrieve，即 ON SELECT DO INSTEAD SELECT 规则——这正是视图的底层实现（每个视图自动有一条无条件 RIR 规则）。

---

## 2. 目录结构

```
src/backend/rewrite/
  rewriteHandler.c     主模块：QueryRewrite / RewriteQuery / fireRIRrules / fireRules
  rewriteManip.c       查询树操作工具：变量重编号、加 qual、改 varlevelsup 等
  rewriteDefine.c      CREATE RULE / 视图 DO INSTEAD 规则的定义
  rewriteRemove.c      DROP RULE
  rewriteSupport.c     规则系统辅助（pg_rewrite 访问、relhasrules 维护）
  rewriteSearchCycle.c WITH RECURSIVE 的 SEARCH/CYCLE 子句展开
  rowsecurity.c        行级安全（RLS）策略的获取与注入
```

规则存储在系统表 **`pg_rewrite`** 中，每条规则的动作是序列化的 `Query` 列表。

---

## 3. 核心数据结构

### 3.1 RewriteRule

视图/规则被加载为 `RewriteRule`（`src/include/rewrite/prs2lock.h`）：

```c
typedef struct RewriteRule
{
    Oid         ruleId;
    CmdType     event;        // 触发事件：CMD_SELECT/INSERT/UPDATE/DELETE
    Node       *qual;         // 规则的 WHERE 条件（条件规则）
    List       *actions;      // 规则动作：要执行的 Query 列表
    bool        isInstead;    // DO INSTEAD？
    char        enabled;      // 启用状态（含 replica/always 模式）
} RewriteRule;
```

一张关系的所有规则汇总在 `RuleLock`（挂在 `RelationData.rd_rules` 上，由 relcache 缓存）。

### 3.2 querySource 标记

`Query.querySource` 在重写中至关重要，用于追踪每个产物 Query 的来源，决定谁来设置命令结果标签（command tag）：

- `QSRC_ORIGINAL`：用户原始查询。
- `QSRC_INSTEAD_RULE` / `QSRC_QUAL_INSTEAD_RULE`：无条件/有条件 INSTEAD 规则产生。
- `QSRC_NON_INSTEAD_RULE`：DO ALSO 规则产生。

---

## 4. 核心算法

### 4.1 QueryRewrite 的三步

`QueryRewrite()`（`rewriteHandler.c:4781`）只处理顶层原始查询（`Assert(querySource == QSRC_ORIGINAL)`），分三步：

```c
List *
QueryRewrite(Query *parsetree)
{
    /* Step 1: 应用所有非 SELECT 规则，可能得到 0 到多个 query */
    querylist = RewriteQuery(parsetree, NIL, 0, 0);

    /* Step 2: 对每个 query 应用 RIR（视图）规则，并打回原始 queryId */
    results = NIL;
    foreach(l, querylist)
    {
        Query *query = fireRIRrules((Query *) lfirst(l), NIL);
        query->queryId = input_query_id;
        results = lappend(results, query);
    }

    /* Step 3: 决定哪个产物 query 设置命令结果标签（canSetTag） */
    ...
    return results;
}
```

**Step 1（`RewriteQuery`）** 处理 INSERT/UPDATE/DELETE 上的规则、可更新视图展开、RLS。它会产生一个 Query 列表（原查询 + DO ALSO 追加的查询，或被 DO INSTEAD 替换）。

**Step 2（`fireRIRrules`）** 递归展开所有视图：扫描 Query 的 rtable，凡是引用视图（带 ON SELECT RIR 规则）的 RTE，调 `ApplyRetrieveRule()`（`rewriteHandler.c:1762`）替换。`fireRIRrules` 会递归进入子查询、CTE、`RTE_FUNCTION` 的内联函数等所有可能含有视图引用的角落。

**Step 3（命令标签归属）** 规则（`rewriteHandler.c` 注释）：
- 若原始查询仍在结果列表中（未被 INSTEAD 替换），由它设标签。
- 否则，与原始查询同类型的**最后一条 INSTEAD 查询**设标签。
- 两者都没有时（全被替换成异类语句），tcop 层用原始未重写查询兜底生成默认标签。

Assert 保证结果列表中至多一个 Query 的 `canSetTag` 为真。

### 4.2 视图展开：ApplyRetrieveRule

核心思路（`ApplyRetrieveRule`，`rewriteHandler.c:1762`）：

1. 取视图的 RIR 规则动作（一条无条件 `SELECT` 子查询 Query）。
2. 把引用视图的那个 `RTE_RELATION` 替换为 `RTE_SUBQUERY`，子查询就是规则动作。
3. 外层查询里所有指向"视图列"的 `Var`，被重映射为引用子查询 targetList 对应表达式（`rewriteManip.c` 的变量替换）。
4. 视图所需的权限信息合并进外层 Query 的 `rteperminfos`，保证权限检查不丢失。

展开后视图对外层完全透明，规划器只看到对基表的子查询，可以做子查询提升（pull-up）等优化。

### 4.3 规则触发：fireRules

`fireRules()`（`rewriteHandler.c:2484`）对某个事件（INSERT/UPDATE/DELETE）遍历目标关系的规则：

- 对每条规则，把规则的 `qual` 加到动作查询上（条件规则）。
- 无条件 INSTEAD 规则会设置 `*instead_flag = true`（`rewriteHandler.c:2511`），告知调用方原始操作被取代。
- DO ALSO 规则的动作被收集进产物列表，与原查询一起返回。

`product_queries` 即"规则产生的派生查询"集合。

### 4.4 可更新视图与 INSTEAD OF 触发器

若对视图执行 DML，重写器先看视图是否有 `INSTEAD OF` 行触发器（`view_has_instead_trigger()`，`rewriteHandler.c:2614`）。有则保留对视图的操作交给触发器；没有且视图"自动可更新"（简单视图）则把 DML 改写到基表（`rewriteTargetView`）。

### 4.5 行级安全（RLS）

`rowsecurity.c` 的 `get_row_security_policies()` 从 `pg_policy` 取出适用于当前角色与命令的策略表达式，重写器把它们用 AND/OR 合并后注入：USING 表达式加到查询的 WHERE（限制可见行），WITH CHECK 表达式作为约束加到写路径。注入后设 `Query.hasRowSecurity = true`。

---

## 5. 设计模式

- **规则即数据**：规则动作是序列化进 `pg_rewrite` 的 `Query` 树。重写是"用存储的查询树替换/扩充当前查询树"，把行为变成可声明、可热更新的数据。
- **视图 = 自动 RIR 规则**：视图不是特殊机制，而是规则系统的一个特例（每个视图建表时自动建一条 ON SELECT DO INSTEAD 规则）。一个机制复用，体现 PostgreSQL 的正交设计。
- **树变换器（walker/mutator）**：`rewriteManip.c` 提供大量基于节点系统的 `query_tree_walker`/`mutator`，统一处理变量重映射、层级调整、qual 注入。
- **来源追踪（querySource）**：用枚举给每个派生查询打来源标签，把"谁负责设命令标签"这种全局语义用局部字段表达。

---

## 6. 架构编排

```
pg_analyze_and_rewrite_*()                   tcop/postgres.c:682
  ├─ parse_analyze_*()  → Query               （见 02-analyzer）
  └─ QueryRewrite(Query)                       rewrite/rewriteHandler.c:4781
       ├─ Step1 RewriteQuery                    非 SELECT 规则 / 可更新视图 / RLS
       │     └─ fireRules / rewriteTargetView / RLS 注入
       ├─ Step2 fireRIRrules（对每个产物）       视图展开
       │     └─ ApplyRetrieveRule               RTE_RELATION → RTE_SUBQUERY
       └─ Step3 canSetTag 归属
       → List<Query>
  → 每个 Query 送入 pg_plan_queries            （见 04-planner-overview）
```

---

## 7. 动手探索

```sql
-- 视图就是规则：查看自动生成的 ON SELECT 规则
CREATE VIEW v AS SELECT * FROM t WHERE active;
SELECT ev_class::regclass, rulename, ev_type, is_instead
FROM pg_rewrite WHERE ev_class = 'v'::regclass;   -- ev_type '1' = SELECT

-- 观察视图被展开成基表子查询
EXPLAIN (VERBOSE) SELECT * FROM v;                 -- 计划里看不到 v，只有 t

-- DO ALSO 规则：一条 INSERT 触发额外写入
CREATE RULE log_ins AS ON INSERT TO t
  DO ALSO INSERT INTO t_log VALUES (new.id, now());

-- RLS
ALTER TABLE t ENABLE ROW LEVEL SECURITY;
CREATE POLICY p ON t USING (owner = current_user);
EXPLAIN (VERBOSE) SELECT * FROM t;                 -- WHERE 里多出 owner = CURRENT_USER
```

调试：在 `QueryRewrite` 出口 `call pprint(results)`，对比展开前后的 rtable —— 视图 RTE 会从 `RTE_RELATION` 变为 `RTE_SUBQUERY`。

---

## 相关模块

- 上游：[02-analyzer](02-analyzer.md)
- 下游：[04-planner-overview](04-planner-overview.md)（子查询提升把展开的视图进一步内联）
- 节点变换工具：[../infra/72-node-system](../infra/72-node-system.md)
