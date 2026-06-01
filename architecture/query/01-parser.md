# Parser — 词法与语法解析

> 源码目录：`src/backend/parser/`（解析部分）
> 核心入口：`raw_parser()` — `src/backend/parser/parser.c:42`
> 上游：`exec_simple_query()` / `pg_parse_query()`　下游：[语义分析 analyzer](02-analyzer.md)

---

## 1. 职责

把一段 **SQL 文本字符串**转换成 **原始语法树（raw parse tree）列表**，即 `List<RawStmt>`。

边界非常清晰，由 `parser.c` 顶部注释（`src/backend/parser/parser.c:6-11`）钉死：

```
 * Note that the grammar is not allowed to perform any table access
 * (since we need to be able to do basic parsing even while inside an
 * aborted transaction).  Therefore, the data structures returned by
 * the grammar are "raw" parsetrees that still need to be analyzed by
 * analyze.c and related files.
```

**为什么解析阶段绝不碰数据库？** `gram.y` 注释（`src/backend/parser/gram.y:24-36`）给出了三条理由：

1. **可在已 abort 的事务中解析**：服务器需要在事务出错后继续读后续命令，直到找到 `ROLLBACK`/`COMMIT`。如果解析要查表，abort 状态下就没法解析了。
2. **多语句字符串整体先解析**：`SET constraint_exclusion TO off; SELECT * FROM foo;` 这一整串会在 `SET` 执行**之前**被 `gram.y` 全部解析。若语法依赖可变状态（如 SET 变量），就会用到错误的状态。
3. 因此一切需要查目录、依赖运行时状态的工作（表名/列名/类型/函数解析）都推迟到 **parse analysis**（见 [02-analyzer](02-analyzer.md)）。

这一职责切分是理解 PostgreSQL 前端的第一原则。

---

## 2. 目录结构

```
src/backend/parser/
  scan.l            Flex 词法规则：SQL 字符串 → token 流
  gram.y            Bison 语法规则（~1.7 万行）：token 流 → RawStmt 节点
  parser.c          raw_parser() 驱动 + base_yylex() 前瞻过滤器
  scansup.c         词法辅助（反斜杠转义、Unicode 转义处理）
  gramparse.h       scanner 与 grammar 之间共享的类型定义
  parse_*.c         语义分析阶段（属于 analyzer，见 02）
```

> 注意：`src/backend/parser/` 目录同时容纳了**解析（scan/gram/parser）** 与**语义分析（parse_*.c）** 两个逻辑阶段。本文只讲前者。

构建期：`scan.l` 由 flex 生成 `scan.c`，`gram.y` 由 bison 生成 `gram.c` + `gram.h`。

---

## 3. 核心数据结构

### 3.1 RawStmt — 语法树的顶层包装

`src/include/nodes/parsenodes.h:2187`：

```c
typedef struct RawStmt
{
    pg_node_attr(no_query_jumble)

    NodeTag     type;
    Node       *stmt;           /* 原始语法树（具体语句节点） */
    ParseLoc    stmt_location;  /* 在原字符串中的起始位置，未知则 -1 */
    ParseLoc    stmt_len;       /* 字节长度；0 表示"到字符串末尾" */
} RawStmt;
```

`stmt_location`/`stmt_len` 记录该语句在整段文本中的切片范围 —— 这就是为什么报错信息能精确定位到出错的子语句、`pg_stat_statements` 能切出单条语句文本。

### 3.2 具体语句节点

`RawStmt.stmt` 指向各种具体语句节点，全部定义在 `src/include/nodes/parsenodes.h`：

- DML：`SelectStmt`、`InsertStmt`、`UpdateStmt`、`DeleteStmt`、`MergeStmt`
- DDL：`CreateStmt`、`AlterTableStmt`、`IndexStmt`、`CreateFunctionStmt` …
- 工具命令：`VacuumStmt`、`CopyStmt`、`ExplainStmt`、`TransactionStmt` …

以 `SelectStmt` 为例，它直接镜像 SQL 子句结构：`targetList`、`fromClause`、`whereClause`、`groupClause`、`havingClause`、`sortClause`、`limitOffset`、`limitCount`、`withClause` 等。这些字段此时全是**未解析的原始形式**：列名只是 `ColumnRef` 字符串，函数调用只是 `FuncCall` 名字，类型只是 `TypeName` 名字 —— 都还没绑定到任何 OID。

### 3.3 节点系统约束

`gram.y:38-42` 的 WARNING 提醒：放进 `List` 的元素必须是 Node（否则打印例程会崩）；赋给 `makeString` 的常量不能被 free。这是因为整个语法树要走[节点系统](../infra/72-node-system.md)的 copy/out/read 通用例程。

---

## 4. 核心算法

### 4.1 raw_parser() 驱动流程

`src/backend/parser/parser.c:42`：

```c
List *
raw_parser(const char *str, RawParseMode mode)
{
    core_yyscan_t yyscanner;
    base_yy_extra_type yyextra;
    int         yyresult;

    /* 初始化 flex 扫描器，挂上关键字表 */
    yyscanner = scanner_init(str, &yyextra.core_yy_extra,
                             &ScanKeywords, ScanKeywordTokens);
    ...
    /* 运行 bison 生成的 LALR(1) 分析器 */
    yyresult = base_yyparse(yyscanner);

    scanner_finish(yyscanner);
    if (yyresult)               /* 语法错误时为非 0 */
        return NIL;
    return yyextra.parsetree;
}
```

`RawParseMode` 支持几种特殊入口：`RAW_PARSE_DEFAULT`（完整 SQL）、`RAW_PARSE_TYPE_NAME`（只解析类型名，给 `cast` 用）、`RAW_PARSE_PLPGSQL_*`（PL/pgSQL 表达式）。不同模式靠在 token 流最前面**注入一个伪 token**（`mode_token[]`）让同一份语法复用为多个入口（`parser.c:58-70`）。

### 4.2 三层流水线：scanner → filter → grammar

```
SQL 文本
  │ core_yylex()   scan.l 生成的纯词法扫描器
  ▼
原始 token 流（IDENT, ICONST, '(', SELECT, NOT, WITH ...）
  │ base_yylex()   parser.c 的前瞻过滤层
  ▼
过滤后的 token 流（NOT_LA, WITH_LA, FORMAT_LA, IDENT ...）
  │ base_yyparse() gram.y 生成的 LALR(1) 分析器
  ▼
RawStmt 列表
```

### 4.3 词法扫描器的"无回溯"设计

`scan.l:12-22` 强调一个性能关键设计：**扫描器永不回溯（no backtrack）**——任何时刻总有一条规则能匹配已消费的输入。flex 的回溯会显著拖慢词法分析（实测几个百分点）。复杂之处主要在浮点数和续行字符串字面量的规则上。改 `scan.l` 后必须用 `flex -b` 验证 `lex.backup` 报告"无需回溯"，Makefile 会自动做这个检查。

### 4.4 base_yylex() —— 把 SQL 的多 token 前瞻压缩成 LALR(1)

标准 SQL 文法有些地方需要**多 token 前瞻**才能决定如何归约，而 bison 是 LALR(1)（只看一个 token）。`base_yylex()`（`parser.c:110`）作为夹在词法器与文法器之间的过滤层，把这些情况就地改写：先取当前 token，若它属于"需要前瞻"的集合，就再读一个 token，根据组合把当前 token 替换成一个专门的 `_LA`（lookahead）变体。

`parser.c:138-161` 列出需要前瞻的 token：`FORMAT`、`NOT`、`NULLS_P`、`WITH`、`WITHOUT`、以及 Unicode 标识符 `UIDENT`/`USCONST`。例如（`parser.c:207-217`）：

```c
case NOT:
    /* 若 NOT 后面跟 BETWEEN/IN/LIKE/ILIKE/SIMILAR，替换为 NOT_LA */
    switch (next_token)
    {
        case BETWEEN:
        case IN_P:
        case LIKE:
        case ILIKE:
        case SIMILAR:
            cur_token = NOT_LA;
            break;
    }
    break;
```

这样文法里 `NOT LIKE`（要作为一个否定运算符）与逻辑 `NOT`（一元布尔否定）就能用不同 token 区分，无需在文法中引入冲突的多 token 规则。`FORMAT JSON`、`WITH ORDINALITY`/`WITH TIME ZONE`、`NULLS FIRST/LAST` 同理。

前瞻读到的 token 被暂存在 `yyextra->lookahead_*`，下次调用 `base_yylex()` 时直接返回（`parser.c:120-128`），所以每个 token 实际只被词法器扫描一次。注释（`parser.c:163-190`）还细致处理了 flex 临时塞入 `'\0'` 的撤销，以保证**错误报告的位置指向当前 token 而非前瞻 token**。

### 4.5 为什么用 filter 而不是在 scan.l 里直接识别多词 token？

`parser.c:96-104` 解释：直接在词法器里识别 `NOT LIKE` 这种多词 token 很麻烦，因为词之间可能有注释，且会重新引入扫描器回溯（性能更差）。filter 层是更简单也更快的方案。同时这里也顺便把 `UIDENT`/`USCONST`（Unicode 转义标识符/字符串）转换回普通 `IDENT`/`SCONST`。

---

## 5. 设计模式

- **生成器 + 驱动**：`scan.l`/`gram.y` 是声明式规则，由 flex/bison 生成命令式 C 代码；`parser.c` 只是薄驱动层。修改语法是改规则文件而非手写状态机。
- **适配器 / 过滤器（base_yylex）**：在两个固定接口（flex 词法器、bison 文法器）之间插入一层做 token 改写，是经典的"无法改两端就在中间夹一层"。
- **不可变产物**：解析阶段产出的 RawStmt 是纯数据、不含 OID、不依赖会话状态，因而可被缓存、可在 abort 事务中产生、可跨阶段安全传递。
- **关键字表分离**：关键字（`ScanKeywords`）独立成表并参与无回溯优化，前端工具（psql 的 `psqlscan.l`、ecpg 的 `pgc.l`）共享同一套词法规则，`scan.l:9-10` 明确要求三者保持同步。

---

## 6. 架构编排：在查询流水线中的位置

```
exec_simple_query()                          tcop/postgres.c:1029
  └─ pg_parse_query(query_string)            tcop/postgres.c:616
       └─ raw_parser(str, RAW_PARSE_DEFAULT) parser/parser.c:42
            ├─ scanner_init / core_yylex       scan.l
            ├─ base_yylex （前瞻过滤）          parser/parser.c:110
            └─ base_yyparse                     gram.y
       → List<RawStmt>
  └─ 对每个 RawStmt：pg_analyze_and_rewrite_*  → 进入 02-analyzer
```

`pg_parse_query()` 还负责在 `log_parser_stats` 打开时统计耗时。返回的 `List<RawStmt>` 随后被逐条送入语义分析与重写。

---

## 7. 动手探索

```sql
-- 看一段 SQL 被切成几条语句、各自的位置
-- （RawStmt.stmt_location/stmt_len 的效果体现在 pg_stat_statements 的语句切分）

-- 触发解析期错误，观察位置精确到子语句
SELECT 1; SELET 2;   -- 第二条报语法错误，位置指向 SELET
```

```bash
# 打开解析器统计
psql -c "SET log_parser_stats = on; SELECT 1;"   # 日志里出现 PARSER STATISTICS

# 阅读生成的状态机（构建后）
less build/src/backend/parser/gram.c
```

调试建议：在 `raw_parser` 下断点，`finish` 后用 `pprint(yyextra.parsetree)`（`call pprint(...)`）打印原始语法树，对照 `nodeToString` 输出理解 RawStmt 结构。

---

## 相关模块

- 下游：[02-analyzer](02-analyzer.md) —— RawStmt → Query
- 节点系统：[../infra/72-node-system](../infra/72-node-system.md) —— makeNode/copyObject/nodeToString
- 错误位置报告：[../infra/71-error-elog](../infra/71-error-elog.md)
