# Executor — 表达式编译与解释执行

> 源码：`src/backend/executor/execExpr.c`（编译）、`execExprInterp.c`（解释）
> 背景文档：`src/backend/executor/README` 的 "Expression Trees / Initialization / Evaluation" 三节
> 配套：[06-executor-overview](06-executor-overview.md)

---

## 1. 职责

执行计划中无处不在的**标量表达式**（WHERE 条件、target list 输出列、join 条件、CHECK 约束、聚合参数等）由本子系统负责求值。它把表达式树（`Expr` 节点：`OpExpr`、`FuncExpr`、`Var`、`Const`、`CaseExpr`…）**编译**成一段线性的"指令序列"，再用解释器（或 JIT 编译的机器码）**执行**。

设计目标（`execExpr.c:13-15`）：编译逻辑与具体执行技术（switch 解释 / computed-goto / JIT）**解耦**——`execExpr.c` 只生成指令，怎么跑由 `execExprInterp.c` 或 JIT 决定。

---

## 2. 为什么不直接递归求值表达式树？

朴素做法是对 `Expr` 树递归 `eval(node)`，但对每行数据都递归一遍代价高昂：分支预测差、函数调用开销大、重复做类型/strict 判断。PostgreSQL 改为**一次编译、多次执行**：

- **初始化期**（每个表达式一次）：`ExecInitExpr` 把树**展平**成 `ExprEvalStep` 数组（线性指令流）。所有"对每行都一样"的决策（哪个函数、是否 strict、目标 slot 偏移）在此固化。
- **运行期**（每行一次）：解释器顺序执行指令数组，无递归、分支可预测，且可被 JIT 编译成原生代码。

这是把"树遍历"降维成"指令流执行"的经典优化，思路类似字节码虚拟机。

---

## 3. 核心数据结构

### 3.1 ExprState — 编译后的表达式

`src/include/nodes/execnodes.h`（`typedef struct ExprState`）：

```c
typedef struct ExprState {
    NodeTag         type;
    uint8           flags;        // EEO_FLAG_IS_QUAL 等
    bool            resnull;      // 标量结果是否为 NULL
    Datum           resvalue;     // 标量结果值
    TupleTableSlot *resultslot;   // 若产出元组（投影），结果放这
    struct ExprEvalStep *steps;   // ★ 指令数组
    ExprStateEvalFunc evalfunc;   // 执行入口（解释器或 JIT 函数）
    Expr           *expr;         // 原始表达式树（调试/JIT 用）
    ...
} ExprState;
```

那些 `#define FIELDNO_EXPRSTATE_RESVALUE 3` 宏暴露字段在结构体中的序号，供 **JIT 代码生成器**按偏移直接读写 `resvalue`/`resnull`——这是解释执行与 JIT 共享同一内存布局的关键。

### 3.2 ExprEvalStep — 一条指令

`src/include/executor/execExpr.h`。每条 step 有一个 opcode（`EEOP_*`）和一个联合体的操作数。代表性 opcode：

| opcode | 含义 |
|--------|------|
| `EEOP_SCAN_VAR` / `EEOP_INNER_VAR` / `EEOP_OUTER_VAR` | 从 scan/inner/outer slot 取某列到 resvalue |
| `EEOP_CONST` | 加载常量 |
| `EEOP_FUNCEXPR` / `EEOP_FUNCEXPR_STRICT` | 调用函数（strict 版先查参数是否 NULL） |
| `EEOP_QUAL` | 布尔短路：结果为假/NULL 立即跳转到末尾 |
| `EEOP_JUMP` / `EEOP_JUMP_IF_NULL` | 控制流（CASE、AND/OR、COALESCE） |
| `EEOP_ASSIGN_TMP` / `EEOP_ASSIGN_SCAN_VAR` | 把结果写进目标 slot 的 `tts_values[]` |
| `EEOP_AGGREF` / `EEOP_WINDOW_FUNC` | 聚合/窗口引用 |
| `EEOP_DONE_RETURN` / `EEOP_DONE_NO_RETURN` | 结束 |

运行期从第一条执行到 `EEOP_DONE_*`（`execExpr.c:10-11`）。

### 3.3 ExprContext — 求值环境

`PlanState.ps_ExprContext`。表达式从这里拿"当前行"：`ecxt_scantuple` / `ecxt_innertuple` / `ecxt_outertuple`（对应 `EEOP_*_VAR` 的取数来源）、外部参数、聚合状态，以及 `ecxt_per_tuple_memory`（每行临时内存域）。

---

## 4. 核心算法

### 4.1 编译：ExecInitExpr → ExecInitExprRec

`ExecInitExpr()`（入口）建空 `ExprState`，调 `ExecInitExprRec()`（`execExpr.c:72`）递归遍历表达式树，对每个节点**追加**相应的 step。"递归"只发生在编译期（每表达式一次），运行期是纯线性的。

关键细节：子表达式的结果必须由**更早的 step** 算好放进约定位置，helper 函数不能自己分发到下一条 step（`README:206-215`）。这保证了 helper 在解释执行与 JIT 下都能复用——它们只算自己那一步，分发由调用方负责。

例如 `a + b * 2`（`OpExpr(+, Var a, OpExpr(*, Var b, Const 2))`）大致编译为：

```
[0] EEOP_SCAN_VAR     b → tmp1
[1] EEOP_CONST        2 → tmp2
[2] EEOP_FUNCEXPR     int4mul(tmp1, tmp2) → tmp_mul
[3] EEOP_SCAN_VAR     a → tmp3
[4] EEOP_FUNCEXPR     int4pl(tmp3, tmp_mul) → resvalue
[5] EEOP_DONE_RETURN
```

### 4.2 选择执行方法：ExecReadyExpr

编译完成后 `ExecReadyExpr()`（`README:200-204`）决定 `evalfunc`：
- 默认指向 `ExecInterpExpr`（`execExprInterp.c` 的大解释器）。
- 极简表达式（如单个 Var、单个 Const）用专门的快速 evalfunc，省掉解释开销。
- 若启用 JIT 且表达式足够"贵"，指向 JIT 编译生成的原生函数（见 4.4）。

### 4.3 解释执行：ExecInterpExpr

`execExprInterp.c` 的 `ExecInterpExpr()` 是一个巨大的 dispatch 循环。两种分发实现：
- **computed goto**（GCC/Clang）：用 `&&label` 标签地址表 + `goto *dispatch[op]`，每条 step 末尾直接跳到下一条，分支预测友好、无循环开销。
- **switch**（可移植回退）：标准 `for(;;) switch(op)`。

`ExecEvalExpr()`（inline，`executor.h`）只是 `return state->evalfunc(state, econtext, isNull)`——上层完全不关心底下是解释还是 JIT。

### 4.4 JIT 编译

`src/backend/jit/`（LLVM 后端在 `jit/llvm/`）。当表达式/元组 deform 代价超过 `jit_above_cost` 等阈值，`llvmjit_expr.c` 按 `ExprState.steps` 生成等价 LLVM IR 再编译成机器码，消除解释循环的分发开销、内联 strict 检查与函数调用。共享的 `FIELDNO_*` 偏移让 JIT 代码与解释器读写同一 `ExprState` 内存。

### 4.5 投影与 qual 的特例

- **投影**（`ExecBuildProjectionInfo`，`README:218-228`）：把 target list 编译成"算出每列 → `EEOP_ASSIGN_TMP` 写进 resultslot 的 `tts_values[]`"。简单 Var 列用 `EEOP_ASSIGN_*_VAR` 一步搞定（省去先算到 resvalue 再搬运的两步）。
- **qual**（`ExecInitQual` → `ExecQual`）：隐式 AND 的条件列表编译成一串 `EEOP_QUAL`，任一为假/NULL 即短路跳到末尾返回 false，实现 WHERE 的提前否决。

---

## 5. 设计模式

- **编译/执行分离（字节码 VM）**：`Expr` 树 → `ExprEvalStep` 指令流 → 解释器/JIT。把"对每行重复的决策"前移到编译期。
- **指令流而非树遍历**：线性 step 数组替代递归，利于分支预测、利于 JIT 顺序代码生成。
- **策略模式（evalfunc）**：同一 `ExprState` 可被解释器、快速特例、JIT 三种 evalfunc 执行，调用方无感。
- **共享 helper（无分发）**：复杂 step 的实现独立成函数，解释与 JIT 共用；约定"不自行分发、子结果靠前序 step 备好"以保持可复用。
- **字段偏移契约（FIELDNO_*）**：用宏暴露结构体字段序号，让生成的机器码与 C 代码共享内存布局。

---

## 6. 架构编排

```
ExecInitNode（每节点初始化时）
  ├─ ExecInitQual(plan->qual)         → ExprState（qual）
  ├─ ExecBuildProjectionInfo(tlist)   → ProjectionInfo{ ExprState }
  └─ ExecInitExpr(各表达式)            execExpr.c
        └─ ExecInitExprRec（递归追加 step）
              → ExecReadyExpr 选 evalfunc（解释 / 特例 / JIT）

运行期（每行）
  ExecQual(qual, econtext)            → ExecEvalExpr → evalfunc
  ExecProject(projInfo)              → ExecEvalExpr → evalfunc
        evalfunc = ExecInterpExpr     execExprInterp.c（computed goto 循环）
                 或 JIT 原生函数        jit/llvm/
```

`ExprContext` 的 `ecxt_per_tuple_memory` 在每行处理后 reset，回收表达式产生的临时 Datum。

---

## 7. 动手探索

```sql
-- 观察 JIT 是否介入（贵查询才触发）
EXPLAIN (ANALYZE, VERBOSE) SELECT sum(a*b+c) FROM big;  -- 输出含 JIT 段
SET jit = off;   -- 关掉对比

-- strict 函数对 NULL 的短路：NULL 输入直接得 NULL，不调函数
SELECT NULL + 1;        -- int4pl 是 strict，整体为 NULL

-- 投影 vs 过滤的表达式分别编译
EXPLAIN VERBOSE SELECT a+1 FROM t WHERE b > 10;
```

调试：在 `ExecInitExprRec` 下断点看 step 如何被追加；在 `ExecInterpExpr` 入口看 `state->steps` 数组内容（`p *state->steps@20`）。

---

## 相关模块

- 执行框架：[06-executor-overview](06-executor-overview.md)
- 类型与 Datum：[../infra/74-datum-types](../infra/74-datum-types.md)
- 节点系统：[../infra/72-node-system](../infra/72-node-system.md)
