# Node 系统 — 统一的树形数据结构框架

> 源码：`src/backend/nodes/`（`gen_node_support.pl` 生成 copy/equal/out/read funcs）
> 头文件：`src/include/nodes/nodes.h`、`nodetags.h`、`parsenodes.h`、`plannodes.h`、`execnodes.h`、`pathnodes.h`

---

## 1. 职责

PostgreSQL 内部所有树形结构——语法树（RawStmt）、查询树（Query）、计划树（Plan）、路径树（Path）、表达式树（Expr）——都建立在统一的 **Node 系统**上。它给这些异构结构提供一套通用操作：创建、深拷贝、结构相等比较、序列化/反序列化。这是把"解析→分析→重写→规划→执行"各阶段的数据结构串起来的底层契约。

---

## 2. 核心约定：第一个字段是 NodeTag

`src/include/nodes/nodes.h:18,129`：

> 每个 node 的第一个字段是 `NodeTag`。

```c
typedef enum NodeTag { T_Invalid = 0, T_Query, T_SeqScan, T_OpExpr, ... } NodeTag;  // nodes.h:26

typedef struct Node { NodeTag type; } Node;   // 抽象基类（只有 tag）
```

任何 node 指针都能安全地当 `Node*` 看，读它的 `type` 知道真实类型。这是 C 里实现**带运行时类型信息的多态**的最小约定——所有通用例程靠 `switch(nodeTag(node))` 分发。

### 关键宏（nodes.h）

```c
#define makeNode(_type_)  ((_type_ *) newNode(sizeof(_type_), T_##_type_))  // 分配+打标 nodes.h:161
#define nodeTag(n)        (((const Node*)(n))->type)
#define IsA(nodeptr, _type_)  (nodeTag(nodeptr) == T_##_type_)             // 类型判断 nodes.h:164
#define castNode(_type_, nodeptr)  ...   // 带 assert 的安全转型           nodes.h:167
```

`NodeTag` 枚举（`nodetags.h`）由代码生成器自动产生，新增一个 node 类型无需手写 tag。

---

## 3. 通用例程（自动生成）

`src/backend/nodes/` 下四组核心函数，**全部由 `gen_node_support.pl` 在构建期生成**（`copyfuncs.c`/`equalfuncs.c`/`outfuncs.c`/`readfuncs.c`，见 `meson.build:20-24`）：

| 函数 | 文件 | 作用 |
|------|------|------|
| `copyObject(node)` | copyfuncs.c | **深拷贝**整棵 node 树 |
| `equal(a, b)` | equalfuncs.c | **结构相等**比较 |
| `nodeToString(node)` | outfuncs.c | **序列化**为文本 |
| `stringToNode(str)` | readfuncs.c | 文本**反序列化**回 node 树 |

### 3.1 代码生成机制

手写这四组函数（覆盖几百种 node、每种几十个字段）既繁琐又易与结构定义不同步。PostgreSQL 的解法：在结构定义（`*.h`）里用 **`pg_node_attr(...)` 属性**标注字段语义，`gen_node_support.pl` 解析这些头文件，**自动生成** copy/equal/out/read 代码。

字段属性示例（见 [../query/02-analyzer](../query/02-analyzer.md) 的 Query 定义）：
- `pg_node_attr(equal_ignore)`：equal 时忽略该字段。
- `pg_node_attr(query_jumble_ignore)`：计算 query 指纹时忽略。
- `pg_node_attr(read_write_ignore)` / `read_as(0)`：序列化时跳过/读为默认。
- `pg_node_attr(copy_as(...))`、`array_size(...)` 等：指导拷贝/数组长度。

改个字段，重新生成即同步——**单一数据源（结构定义）驱动多份样板代码**。

---

## 4. 典型用途

### 4.1 copyObject —— 计划缓存与安全传递

`copyObject` 深拷贝让一棵树可被安全复用而不怕被改：plancache 缓存解析树/计划树后，每次执行可拷一份用（见 [../catalog/41-caches](../catalog/41-caches.md)）；重写、规划各阶段也常拷贝子树再变换。

### 4.2 nodeToString/stringToNode —— 跨进程传 Plan

**并行查询**的基石：leader 把 `PlannedStmt` 用 `nodeToString` 序列化成文本，放进 DSM，worker 用 `stringToNode` 反序列化重建出同一棵计划树（见 [../process/53-parallel-query](../process/53-parallel-query.md)）。多进程无法共享指针，只能序列化传输。视图定义、规则动作也以 `nodeToString` 形式存进系统表（`pg_rewrite.ev_action`，见 [../query/03-rewriter](../query/03-rewriter.md)）。

### 4.3 equal —— 表达式比较

优化器判断两个表达式是否等价（如等价类成员、去重 target list）用 `equal`。

---

## 5. 节点遍历：walker 与 mutator

`src/backend/nodes/nodeFuncs.c` 提供通用遍历框架：
- **`expression_tree_walker(node, walker, ctx)`** / **`query_tree_walker`**：递归访问每个子节点，对每个调用户的 `walker` 回调（只读分析，如"找出所有 Var"、"是否含聚合"）。
- **`expression_tree_mutator`** / **`query_tree_mutator`**：递归并允许**替换**节点，返回变换后的新树（如重写器的变量重映射、规划器的常量折叠）。

用户只需写"对感兴趣的 node 类型怎么处理"，递归下降的样板由框架包办。这让"对任意查询树做某种变换/收集"变得简洁——遍及 analyzer、rewriter、planner。

---

## 6. 设计模式

- **统一基类 + 运行时标签（NodeTag）**：用"第一个字段是 tag"的最小约定，在 C 中实现带 RTTI 的多态，使一套通用例程能处理上百种异构结构。
- **属性驱动的代码生成**：在结构定义处用 `pg_node_attr` 标注语义，单一来源生成 copy/equal/out/read，根除样板代码与结构定义的不同步。
- **序列化即可移植性**：`nodeToString/stringToNode` 把指针树转成可存储/可传输的文本，支撑并行查询、视图存储、规则系统。
- **walker/mutator 模板方法**：把"递归下降"与"对某节点做什么"分离，遍历骨架复用，业务逻辑只写回调——贯穿前端各阶段的树变换。
- **深拷贝保不变性**：`copyObject` 让缓存树可安全复用，各阶段变换不污染上游。

---

## 7. 架构编排

```
结构定义（parsenodes.h/plannodes.h/...，含 pg_node_attr 标注）
   │ 构建期
   └─ gen_node_support.pl ──> nodetags.h + copyfuncs/equalfuncs/outfuncs/readfuncs.c

运行期通用操作：
  makeNode(T)        创建并打标
  copyObject(n)      深拷贝（plancache、各阶段变换）       catalog/41
  equal(a,b)         结构比较（优化器）
  nodeToString(n)    序列化（并行 DSM、视图/规则存储）      process/53, query/03
  stringToNode(s)    反序列化（worker 重建计划）
  expression_tree_walker/mutator  递归遍历/变换            query/02,03,04
```

---

## 8. 动手探索

```sql
-- nodeToString 的产物：视图/规则以序列化 node 树存储
SELECT ev_action FROM pg_rewrite WHERE ev_class = 'some_view'::regclass;  -- 一长串 {QUERY ...}

-- 打印各阶段 node 树到日志
SET debug_print_parse = on;     -- 解析树
SET debug_print_rewritten = on; -- 重写后
SET debug_print_plan = on;      -- 计划树
SET client_min_messages = debug1;
SELECT 1;
```

调试：gdb 里 `call pprint(node)` 对任意 node 指针打印其树（基于 outfuncs）；`p nodeTag(node)` 看类型。

---

## 相关模块

- 各阶段产物：[../query/01-parser](../query/01-parser.md)~[04](../query/04-planner-overview.md)
- 跨进程传输：[../process/53-parallel-query](../process/53-parallel-query.md)
- 规则/视图存储：[../query/03-rewriter](../query/03-rewriter.md)
- 计划缓存：[../catalog/41-caches](../catalog/41-caches.md)
- 集合类型：[73-collections](73-collections.md)
