# Datum 与类型系统

> 源码：`src/include/postgres.h`、`src/backend/utils/adt/`、`src/include/fmgr.h`
> 系统表：`pg_type` / `pg_proc` / `pg_cast`（见 [../catalog/40-catalog](../catalog/40-catalog.md)）

---

## 1. 职责

PostgreSQL 的类型系统是**可扩展**的：内置类型与用户自定义类型平起平坐，都靠系统表描述、靠函数实现行为。本模块讲两个底层支柱：
- **Datum**：统一的"一个值"的运行时表示，让引擎以类型无关的方式传递任意类型的值。
- **fmgr（函数管理器）**：以统一调用约定调用任意类型的函数（输入输出、运算符、cast）。

这套机制让"加一个新类型"= 往系统表插几行 + 写几个函数，无需改引擎。

---

## 2. Datum —— 统一的值容器

`src/include/postgres.h:70`：

```c
typedef uint64_t Datum;   // 与指针等宽的整数（64 位平台 8 字节）
```

`Datum` 是一个机器字，按类型的"传递方式"承载值：

### 2.1 pass-by-value vs pass-by-reference

`pg_type.typbyval` 决定：
- **传值（by-value）**：值 ≤ Datum 宽度（int4、int8、bool、float8、oid…）→ 值**直接编码进 Datum** 的位里。
- **传引用（by-reference）**：值更大或变长（text、numeric、数组、自定义大类型）→ Datum 里存的是**指向值的指针**。

转换宏（`postgres.h:198`）屏蔽差异：

```c
int32  x  = DatumGetInt32(d);     Datum d = Int32GetDatum(x);    // 传值：位操作
text  *t  = DatumGetTextP(d);     Datum d = PointerGetDatum(t);  // 传引用：取/存指针
```

引擎（executor、表达式、slot 的 `tts_values[]`）统一用 `Datum` 搬运值，配合 `tts_isnull[]` 表示 NULL，完全不关心具体类型——类型差异只在调用类型相关函数时才展开。

### 2.2 变长类型与 TOAST

变长类型（`typlen = -1`，如 text/bytea/numeric/数组）以 `varlena` 表示：开头是长度头，后跟数据。大值可能被压缩或外置存储（TOAST，见 [../storage/15-page-layout](../storage/15-page-layout.md)）。函数拿到这种 Datum 前常需 `PG_DETOAST_DATUM` 解压/取回。`typlen = -2` 是 C 字符串（cstring）。

---

## 3. pg_type —— 类型的元数据

`pg_type` 一行描述一个类型的全部属性（见 [../catalog/40-catalog](../catalog/40-catalog.md)）：

| 字段 | 含义 |
|------|------|
| `typlen` | 长度：正数=定长、-1=变长(varlena)、-2=cstring |
| `typbyval` | 传值还是传引用 |
| `typalign` | 对齐要求 |
| `typinput` / `typoutput` | 文本 ↔ 内部表示的 I/O 函数（`'123'` ↔ int4） |
| `typreceive` / `typsend` | 二进制 I/O |
| `typelem` / `typarray` | 数组元素/该类型的数组类型 |
| `typrelid` | 复合类型对应的 pg_class |

类型的"行为"全在这些指向 `pg_proc` 的函数里。引擎想把一个值转成文本，就调它的 `typoutput`——对所有类型一视同仁。

---

## 4. fmgr —— 函数管理器

`src/include/fmgr.h`。所有 SQL 可见函数（包括运算符的实现、类型 I/O、cast）都遵循**统一的 C 调用约定**，由 fmgr 统一调用：

```c
Datum some_func(PG_FUNCTION_ARGS)        // 宏展开为 (FunctionCallInfo fcinfo)
{
    int32 a = PG_GETARG_INT32(0);        // 取第 0 个参数
    text *b = PG_GETARG_TEXT_PP(1);
    if (PG_ARGISNULL(0)) PG_RETURN_NULL();
    ...
    PG_RETURN_INT32(result);             // 返回 Datum
}
```

`FunctionCallInfo`（fcinfo）携带参数 Datum 数组、各参数 isnull、collation、函数元信息等。`FunctionCallN`/`OidFunctionCall` 等按 OID 查 `pg_proc` 拿到 C 函数指针（`FmgrInfo` 缓存）再调用。表达式执行（见 [../query/07-executor-expr](../query/07-executor-expr.md)）的 `EEOP_FUNCEXPR` 最终就是走 fmgr 调用。

### 4.1 strict 与 NULL

`pg_proc.proisstrict`：strict 函数任一参数为 NULL 则结果直接 NULL，根本不调函数体（由表达式层 `EEOP_FUNCEXPR_STRICT` 短路，见 [../query/07-executor-expr](../query/07-executor-expr.md)）。非 strict 函数自己处理 NULL（用 `PG_ARGISNULL`）。

---

## 5. 运算符、cast 与重载

- **运算符**（`pg_operator`）：`a + b` 解析为某个 operator，其 `oprcode` 指向 `pg_proc` 里的实现函数（如 `int4pl`）。运算符就是有特殊语法的函数。
- **cast**（`pg_cast`）：类型转换规则，`castfunc` 指向转换函数，`castcontext` 标明隐式/赋值/显式。analyzer 据此插入类型强制（见 [../query/02-analyzer](../query/02-analyzer.md)）。
- **重载**：同名函数/运算符可有多个不同参数类型的版本；分析阶段按实参类型做"最佳匹配"决议（见 [../query/02-analyzer](../query/02-analyzer.md) §4.4）。

---

## 6. 设计模式

- **统一值容器（Datum）**：用一个机器字 + 传值/传引用约定承载任意类型，让引擎以类型无关方式搬运值，类型差异延迟到调用类型函数时才展开——可扩展类型系统的运行时基石。
- **元数据 + 函数指针描述类型（pg_type → pg_proc）**：类型的所有行为外包给系统表里登记的函数，新增类型无需改引擎，内置与自定义类型同构。
- **统一调用约定（fmgr / PG_FUNCTION_ARGS）**：所有 SQL 函数遵循同一 C 签名，使按 OID 动态调用、参数/NULL/collation 处理标准化，支撑表达式执行与重载。
- **strict 短路**：把"NULL 输入即 NULL 输出"上提到调用层统一处理，函数体免写 NULL 样板。
- **运算符即函数、cast 即函数**：把运算符与类型转换都归约为登记在系统表里的函数，机制最小化、正交。
- **varlena + TOAST 透明大值**：变长类型用自描述长度头 + 按需 detoast，让超大值对函数大体透明。

---

## 7. 架构编排

```
SQL 值 '123'::int4
  typinput(int4in) ──文本→内部──> int32 → Int32GetDatum → Datum（值编码进位）
执行期：
  slot.tts_values[i] = Datum；tts_isnull[i] = NULL 标志        见 query/06
  表达式 a+b：EEOP_FUNCEXPR → fmgr → int4pl(PG_FUNCTION_ARGS)   见 query/07
        └─ pg_operator(+).oprcode → pg_proc(int4pl) → FmgrInfo 缓存
输出：
  typoutput(int4out) ──内部→文本──> '123' → 经 printtup 发客户端
类型转换：analyzer 据 pg_cast 插入 cast 函数                    见 query/02
```

---

## 8. 动手探索

```sql
-- 类型元数据
SELECT typname, typlen, typbyval, typalign,
       typinput::regproc, typoutput::regproc, typarray::regtype
FROM pg_type WHERE typname IN ('int4','text','numeric','bool');

-- 运算符 = 函数
SELECT oprname, oprcode::regproc FROM pg_operator WHERE oprname='+' LIMIT 5;

-- cast 规则
SELECT castsource::regtype, casttarget::regtype, castcontext, castfunc::regproc
FROM pg_cast WHERE castsource='int4'::regtype;

-- 值大小（传值 vs 传引用 / TOAST）
SELECT pg_column_size(1::int4), pg_column_size('hi'::text), pg_column_size(repeat('x',10000));
```

---

## 相关模块

- 类型元数据存储：[../catalog/40-catalog](../catalog/40-catalog.md)
- 表达式调用 fmgr：[../query/07-executor-expr](../query/07-executor-expr.md)
- 类型决议与 cast：[../query/02-analyzer](../query/02-analyzer.md)
- 大值存储：[../storage/15-page-layout](../storage/15-page-layout.md)
- slot 里的 Datum：[../query/06-executor-overview](../query/06-executor-overview.md)
