# Analyzer — 语义分析

> 源码目录：`src/backend/parser/`（`analyze.c` + `parse_*.c`）
> 核心入口：`parse_analyze_fixedparams()` — `src/backend/parser/analyze.c:127`
> 上游：[01-parser](01-parser.md)（RawStmt）　下游：[03-rewriter](03-rewriter.md)（Query）

---

## 1. 职责

把语法正确但**未绑定语义**的 `RawStmt`，转换成完全解析的 **`Query`** 查询树。这是第一个**会访问数据库**的阶段，承担：

- **名称解析**：表名 → 关系 OID、列名 → 列号、类型名 → 类型 OID、函数/运算符名 → pg_proc/pg_operator OID。
- **类型检查与强制转换**：表达式类型推导，插入隐式 cast（coercion）。
- **语义校验**：GROUP BY 是否覆盖所有非聚合列、聚合/窗口函数位置是否合法、子查询返回列数是否匹配等。
- **结构规范化**：把 SQL 子句翻译为 `Query` 的标准字段（rtable、jointree、targetList…）。

`RawStmt` 是"SQL 长什么样"，`Query` 是"SQL 在语义上要做什么"。

---

## 2. 目录结构

`src/backend/parser/` 中属于语义分析的文件：

```
analyze.c          parse_analyze_*() 入口；按语句类型分发到 transformXxxStmt()
parse_clause.c     FROM / WHERE / GROUP BY / ORDER BY / LIMIT / WINDOW 子句
parse_relation.c   表/列名解析，构建并查找 RangeTblEntry（RTE）
parse_expr.c       表达式树转换（运算符、CASE、子查询、数组等）
parse_func.c       函数调用解析（名称查找 + 重载决议 + 权限检查）
parse_oper.c       运算符解析与重载决议
parse_type.c       类型名 → 类型 OID
parse_coerce.c     类型强制转换（隐式/赋值/显式 cast）
parse_collate.c    排序规则（collation）赋值
parse_target.c     SELECT 目标列展开（`*` 展开、列别名）
parse_agg.c        聚合/窗口函数语义校验
parse_cte.c        WITH（CTE）与 WITH RECURSIVE
parse_param.c      $n 参数处理
parse_merge.c      MERGE 语句
parse_utilcmd.c    CREATE/ALTER 等 utility 命令的 transform（拆约束、补默认值）
parse_node.c       节点创建辅助、ParseState 操作
parse_jsontable.c  JSON_TABLE
```

---

## 3. 核心数据结构

### 3.1 Query — 分析阶段的产物

`src/include/nodes/parsenodes.h`（`typedef struct Query`）。关键字段（省略大量布尔标志位）：

```c
typedef struct Query
{
    NodeTag     type;
    CmdType     commandType;     // CMD_SELECT/INSERT/UPDATE/DELETE/MERGE/UTILITY
    QuerySource querySource;     // 来源：原始 SQL / 规则展开 / ...
    int64       queryId;         // query jumble 指纹（pg_stat_statements 用）
    Node       *utilityStmt;     // commandType==CMD_UTILITY 时挂原始 utility 节点

    int         resultRelation;  // INSERT/UPDATE/DELETE/MERGE 的目标表在 rtable 中的下标；SELECT 为 0

    /* 一组语义标志，分析阶段填充，供后续阶段快速判断 */
    bool        hasAggs;         // tlist/having 中有聚合
    bool        hasWindowFuncs;  // 有窗口函数
    bool        hasSubLinks;     // 有子查询 SubLink
    bool        hasRecursive;    // WITH RECURSIVE
    bool        hasRowSecurity;  // 重写器加过 RLS 策略
    ...

    List       *cteList;         // WITH 列表（CommonTableExpr）
    List       *rtable;          // RangeTblEntry 列表（FROM 中每一项）
    List       *rteperminfos;    // 需要权限检查的 RTE 的权限信息
    FromExpr   *jointree;        // FROM + WHERE 的连接树
    List       *targetList;      // TargetEntry 列表（SELECT 输出列 / DML 赋值列）
    OnConflictExpr *onConflict;  // ON CONFLICT 子句
    List       *groupClause;     // GROUP BY
    List       *distinctClause;  // DISTINCT [ON]
    List       *sortClause;      // ORDER BY
    List       *windowClause;    // WINDOW
    Node       *limitOffset;
    Node       *limitCount;
    List       *rowMarks;        // FOR UPDATE/SHARE
    List       *returningList;   // RETURNING
    ...
} Query;
```

注意那些 `pg_node_attr(query_jumble_ignore)` 标记：它们告诉 query-jumble 生成器在计算 `queryId`（用于 `pg_stat_statements` 归并"同形"查询）时忽略该字段。这是节点系统属性驱动代码生成的典型应用（见 [../infra/72-node-system](../infra/72-node-system.md)）。

### 3.2 RangeTblEntry（RTE）— FROM 项的统一表示

`src/include/nodes/parsenodes.h`。一条 RTE 表示 FROM 中的一个数据来源，`rtekind` 区分类型：

| rtekind | 含义 | 关键字段 |
|---------|------|---------|
| `RTE_RELATION` | 普通表/索引/物化视图 | `relid`、`relkind`、`rellockmode` |
| `RTE_SUBQUERY` | 子查询 | `subquery`（嵌套 Query） |
| `RTE_JOIN` | JOIN 结果 | `jointype`、`joinaliasvars` |
| `RTE_FUNCTION` | 函数调用 FROM | `functions` |
| `RTE_VALUES` | VALUES 列表 | `values_lists` |
| `RTE_CTE` | 引用 WITH 项 | `ctename`、`ctelevelsup` |
| `RTE_GROUP` | GROUP BY 表达式承载（新） | `groupexprs` |

`rtable` 是一个**扁平数组式列表**：所有 RTE 用从 1 开始的整数下标引用（`Var.varno`、`resultRelation` 等都是这个下标），这是整个查询树里最重要的间接寻址约定。

### 3.3 ParseState — 贯穿分析的上下文

`src/include/parser/parse_node.h`（`typedef struct ParseState`）。它像一个"分析期会话"，在所有 `transformXxx` 函数间传递：

- `p_rtable`：正在构建的 RTE 列表。
- `p_namespace`：当前可见的列名空间（哪些 RTE 的列可以被无限定地引用）。
- `p_parent_cte` / `p_ctenamespace`：CTE 可见性。
- `p_expr_kind`：当前正在分析的表达式上下文（如 `EXPR_KIND_WHERE`、`EXPR_KIND_GROUP_BY`），用于产生精确报错（"聚合函数不能出现在 WHERE 中"）。
- `p_paramref_hook` 等钩子：让 PL/pgSQL 等调用方注入自定义的参数/列解析逻辑。
- 父 ParseState 链：支持子查询逐层向上查找列。

---

## 4. 核心算法

### 4.1 入口与分发

`parse_analyze_fixedparams()`（`analyze.c:127`）创建 `ParseState`，记录源文本和固定参数类型，然后调 `transformTopLevelStmt()`（`analyze.c:271`）→ `transformStmt()`（`analyze.c:334`）。`transformStmt()` 是一个按 `nodeTag(parseTree)` 分发的大 switch：

```
SelectStmt  → transformSelectStmt() / transformSetOperationStmt()
InsertStmt  → transformInsertStmt()
UpdateStmt  → transformUpdateStmt()
DeleteStmt  → transformDeleteStmt()
MergeStmt   → transformMergeStmt()
其它（DDL/utility）→ 多数原样包进 Query.utilityStmt（CMD_UTILITY），
                     少数（CREATE TABLE AS、DECLARE CURSOR 等）需要进一步 transform
```

变长参数版本 `parse_analyze_varparams()`（`analyze.c:167`）用于 `PREPARE` 不指定全部参数类型的情况，分析后能反推 `$n` 的类型。

### 4.2 transformSelectStmt 的工作顺序

`transformSelectStmt()`（`analyze.c:1747`）的顺序很关键，因为后面的子句依赖前面建立的命名空间：

1. **FROM**：`transformFromClause()` → 对每个 FROM 项调 `transformFromClauseItem()`，为表打开 relation、建 RTE、加入 `p_namespace`。JOIN 递归处理并产生 `RTE_JOIN`。
2. **target list**：`transformTargetList()` 展开 `*`、解析每个输出表达式、生成 `TargetEntry`。
3. **WHERE**：`transformWhereClause()` 解析为布尔表达式，挂到 `jointree->quals`。
4. **GROUP BY / HAVING**：`transformGroupClause()`，并由 `parse_agg.c` 校验。
5. **DISTINCT / ORDER BY / WINDOW / LIMIT**：依次转换，ORDER BY/DISTINCT 可引用 target list 的输出别名。
6. 最后 `parseCheckAggregates()` 做聚合一致性总校验。

### 4.3 名称解析：列名 → Var

无限定列名 `c` 的解析（`parse_relation.c` 的 `colNameToVar()` / `scanRTEForColumn()`）：遍历 `p_namespace` 中的 RTE，看哪个 RTE 暴露了名为 `c` 的列；若恰好一个则解析成功，多个则报"column reference is ambiguous"，零个则向父 ParseState 上溯。限定列名 `t.c` 先按别名 `t` 定位 RTE，再取列。最终产物是 `Var { varno=RTE下标, varattno=列号, vartype, ... }`。

### 4.4 表达式转换与类型推导

`transformExpr()`（`parse_expr.c`）递归下降处理每种表达式节点：

- `ColumnRef` → `Var`（名称解析，见上）。
- `A_Const` → `Const`（带类型 OID）。
- `FuncCall` → `func_get_detail()` 做**函数重载决议**：按实参类型在候选函数中找最佳匹配，必要时插入隐式 cast（`parse_func.c`）。
- `A_Expr`（运算符）→ `oper()` 做运算符重载决议（`parse_oper.c`）。
- `SubLink`（子查询）→ 递归 `parse_analyze` 子查询，设 `Query.hasSubLinks`。
- 类型不匹配时由 `parse_coerce.c` 的 `coerce_type()` 按 `pg_cast` 决定能否隐式转换；不能则报类型错误。

类型决议遵循"最佳匹配"规则（详见文档 *SQL Language → Type Conversion*），核心是 `func_select_candidate()` 的多轮淘汰。

### 4.5 聚合与 GROUP BY 校验

`parse_agg.c` 的 `check_ungrouped_columns_walker()` 遍历 target list 和 HAVING：凡是出现的列，要么在某个聚合函数的参数内，要么必须出现在 GROUP BY 中，否则报 *"column must appear in the GROUP BY clause or be used in an aggregate function"*。窗口函数则校验不能嵌套、不能出现在 WHERE/GROUP BY 中。

---

## 5. 设计模式

- **Transform 递归下降**：每种语句/子句/表达式一个 `transformXxx` 函数，结构与文法对称。这是手写递归下降"语义动作"的组织方式。
- **上下文对象（ParseState）**：避免在几十个函数间传一长串参数，用一个可变上下文承载命名空间、参数、表达式种类等。
- **钩子（hook）扩展点**：`p_paramref_hook`、`p_coerce_param_hook` 等让 PL/pgSQL、SPI 注入自定义解析，而不污染核心。
- **重载决议即"最佳匹配评分"**：函数/运算符/类型转换统一用候选集 + 逐轮淘汰，是 PostgreSQL 类型系统可扩展性的根基。
- **扁平 RTE + 整数下标**：用从 1 开始的下标在整棵树里引用关系，贯穿 analyzer/planner/executor，是全内核的统一寻址约定。

---

## 6. 架构编排

```
pg_analyze_and_rewrite_fixedparams()         tcop/postgres.c:682
  └─ parse_analyze_fixedparams(RawStmt, ...)  parser/analyze.c:127
       ├─ make_parsestate()                    建 ParseState
       └─ transformTopLevelStmt()              analyze.c:271
            └─ transformStmt() 按类型分发        analyze.c:334
                 └─ transformSelectStmt() 等     analyze.c:1747
                      ├─ transformFromClause     parse_clause.c → 建 RTE
                      ├─ transformTargetList     parse_target.c
                      ├─ transformWhereClause    parse_clause.c
                      ├─ transformExpr (递归)     parse_expr.c / parse_func.c / parse_oper.c / parse_coerce.c
                      └─ parseCheckAggregates    parse_agg.c
       → Query
  └─ QueryRewrite(Query)                        → 进入 03-rewriter
```

分析阶段为每张被引用的表加 `AccessShareLock`（或 DML 对应更强的锁），锁一直持有到事务结束，保证从分析到执行期间表结构不被并发 DDL 改变。

---

## 7. 动手探索

```sql
-- 观察类型推导与隐式 cast 的插入
EXPLAIN VERBOSE SELECT 1 + 2.0;        -- integer 被隐式转 numeric

-- 触发各类语义错误，理解校验点
SELECT a, count(*) FROM t;             -- a 必须在 GROUP BY 或聚合内
SELECT t.x FROM (SELECT 1) t(y);       -- 列 x 不存在
SELECT count(count(*)) FROM t;         -- 聚合不能嵌套
```

调试：在 `transformStmt` 下断点，逐步进入 `transformSelectStmt`，在返回处 `call pprint(query)` 查看完整 Query 树；对比 `Var` 的 `varno/varattno` 与 `rtable` 下标的对应关系。

---

## 相关模块

- 上游：[01-parser](01-parser.md)
- 下游：[03-rewriter](03-rewriter.md)、[04-planner-overview](04-planner-overview.md)
- 类型系统：[../infra/74-datum-types](../infra/74-datum-types.md)
- 目录访问：[../catalog/41-caches](../catalog/41-caches.md)（syscache 支撑名称解析）
