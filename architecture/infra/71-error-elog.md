# 错误处理 — ereport / elog 与异常模型

> 源码：`src/backend/utils/error/elog.c`
> 头文件：`src/include/utils/elog.h`

---

## 1. 职责

提供统一的错误报告与**异常控制流**。C 没有异常，PostgreSQL 用 `setjmp/longjmp` 自建了一套：`ereport(ERROR)` 能从任意深度的调用栈"抛出"，跳回最近的捕获点（通常是事务边界或 `PG_TRY`），并配合 MemoryContext（见 [70-memory-context](70-memory-context.md)）自动清理资源。同时负责把消息按级别输出到客户端与服务器日志。

---

## 2. 错误级别

`elog.h`，从轻到重：

| 级别 | 行为 |
|------|------|
| `DEBUG1..5` | 调试信息 |
| `LOG` / `INFO` / `NOTICE` / `WARNING` | 信息性，不中断流程 |
| `ERROR` | **抛出异常**：longjmp 回最近捕获点，回滚当前（子）事务 |
| `FATAL` | 终止当前 backend 连接 |
| `PANIC` | 终止整个服务器（所有进程重置） |

`ERROR` 是分水岭：`< ERROR` 的只是输出消息后继续；`>= ERROR` 的会改变控制流。

---

## 3. 报告 API：ereport

`elog.h:166` 的 `ereport`：

```c
ereport(ERROR,
        (errcode(ERRCODE_UNDEFINED_TABLE),
         errmsg("relation \"%s\" does not exist", relname),
         errhint("...")));
```

`ereport(level, rest...)` 是宏：`errcode()`/`errmsg()`/`errdetail()`/`errhint()`/`errposition()` 等辅助函数往一个"当前错误数据"结构里填字段，最后由 `errfinish()` 决定输出与是否 longjmp。`elog(level, fmt, ...)` 是简化版（只设 message，给内部错误用）。

`errcode` 是 SQLSTATE（5 位），客户端据此程序化判断错误类型（如 `40001` 序列化失败、`23505` 唯一约束冲突）。

---

## 4. 异常控制流：PG_TRY / PG_CATCH

`elog.h:324` 一带的宏，基于 `sigsetjmp`：

```c
PG_TRY();
{
    // 可能 ereport(ERROR) 的代码
}
PG_CATCH();
{
    // 清理（释放非内存资源、回滚部分操作）
    PG_RE_THROW();   // 通常重新抛出，让上层继续处理
}
PG_END_TRY();
```

机制：
- `PG_TRY` 把当前 `PG_exception_stack`（一个 `sigjmp_buf*` 链）保存、设为本块。
- `ereport(ERROR)` 的 `errfinish` 调 `siglongjmp(*PG_exception_stack, 1)`，跳回最近 `PG_TRY` 的 `sigsetjmp`，进入 `PG_CATCH`。
- `PG_CATCH` 块负责释放**非内存资源**（文件句柄、锁等需手工处理的），内存由 MemoryContext 自动回收。
- `PG_RE_THROW` 继续向上 longjmp。
- `PG_FINALLY`（与 PG_CATCH 二选一，`elog.h:356`）：无论是否出错都执行清理。

约束（`elog.h:372-382`）：在 PG_TRY 块里修改、又在 PG_CATCH 用的局部变量要加 `volatile`，否则 longjmp 后值可能不确定（编译器优化所致）。

---

## 5. 顶层捕获：事务边界

最重要的捕获点不在某个 PG_TRY，而在 **PostgresMain 的主循环顶部**（`postgres.c`）：

```c
if (sigsetjmp(local_sigjmp_buf, 1) != 0) {
    // 任何未被内层 PG_CATCH 处理的 ERROR 跳到这里
    EmitErrorReport();              // 发错误给客户端 + 写日志
    AbortCurrentTransaction();      // 回滚当前事务
    ... 重置 MessageContext 等 ...
}
PG_exception_stack = &local_sigjmp_buf;
for (;;) { /* 读命令、执行 */ }
```

于是任何深处的 `ereport(ERROR)`，只要没被内层捕获，最终都跳回这里：**回滚事务、清理内存、发错误码给客户端、继续读下一条命令**。这就是为什么一条语句出错不会拖垮整个连接——`AbortCurrentTransaction` 删除事务上下文（[70-memory-context](70-memory-context.md)）+ 释放锁（[../concurrency/22-heavyweight-lock](../concurrency/22-heavyweight-lock.md)）+ 释放 LWLock（[../concurrency/21-lwlock](../concurrency/21-lwlock.md)）一并兜底。

子事务（savepoint / PL/pgSQL EXCEPTION 块，见 [../transaction/30-xact](../transaction/30-xact.md)）则在更内层设捕获点，使错误只回滚子事务。

---

## 6. 错误上下文栈：error_context_stack

`elog.h:318` 的 `error_context_stack`：一个回调链，让各层在报错时**追加定位信息**而不改报错点代码。例如执行 PL/pgSQL 函数时压入一个回调，报错时它追加 "PL/pgSQL function f() line 12 at ..."；COPY 时追加 "COPY t, line 5"。`errfinish` 输出错误时遍历这个栈，逐层调用回调拼出完整的 CONTEXT 信息。压栈/弹栈由各模块用局部 `ErrorContextCallback` 管理。

---

## 7. 设计模式

- **C 上的异常机制（setjmp/longjmp）**：用 `sigsetjmp` + 异常栈 + `ereport(ERROR)` 在无异常的 C 里实现"从任意深度抛出、跳回捕获点"，是整个内核错误处理的骨架。
- **异常 + 区域内存 = 自动清理**：把内存清理交给 MemoryContext（出错即重置上下文），PG_CATCH 只管非内存资源，二者配合让"出错自动回收"几乎免费（见 [70](70-memory-context.md)）。
- **顶层兜底捕获**：在主循环设最终捕获点，保证任何漏网的 ERROR 都被收敛为"回滚 + 报错 + 继续"，连接级鲁棒性。
- **结构化错误（SQLSTATE + 多字段）**：errcode/msg/detail/hint 分离，机器可判别、人可读，国际化友好。
- **旁路上下文栈（error_context_stack）**：用回调链在报错时按需注入定位信息，报错点与上下文采集解耦。
- **级别决定控制流**：用 `elevel` 统一表达"信息/异常/致命/灾难"，`< ERROR` 继续、`>= ERROR` 改流，简洁分层。

---

## 8. 架构编排

```
任意深度代码
  ereport(ERROR, (...))                         elog.h:166
    └─ errfinish → siglongjmp(*PG_exception_stack)
         ├─ 最近 PG_TRY/PG_CATCH 捕获 → 局部清理 → PG_RE_THROW ↑
         └─ 无内层捕获 → 跳到子事务捕获点（PL EXCEPTION/savepoint）
              └─ 或最终跳到 PostgresMain 顶层 sigsetjmp
                   ├─ EmitErrorReport（客户端 + 日志，遍历 error_context_stack）
                   ├─ AbortCurrentTransaction（重置事务上下文、放锁/LWLock）
                   └─ 继续读下一条命令
信息级（< ERROR）：直接输出，不改控制流
```

---

## 9. 动手探索

```sql
-- 触发各类错误看 SQLSTATE
SELECT 1/0;                       -- 22012 division_by_zero
INSERT INTO t VALUES(dup_pk);     -- 23505 unique_violation
SELECT * FROM no_such_table;      -- 42P01 undefined_table

-- CONTEXT（error_context_stack 的产物）
CREATE FUNCTION f() RETURNS int LANGUAGE plpgsql AS $$ BEGIN RETURN 1/0; END $$;
SELECT f();   -- 错误带 "CONTEXT: PL/pgSQL function f() line 1 ..."

-- PL/pgSQL EXCEPTION = 子事务捕获
DO $$ BEGIN PERFORM 1/0; EXCEPTION WHEN division_by_zero THEN RAISE NOTICE 'caught'; END $$;

SHOW log_min_messages; SHOW client_min_messages;
```

---

## 相关模块

- 内存自动清理：[70-memory-context](70-memory-context.md)
- 事务回滚：[../transaction/30-xact](../transaction/30-xact.md)
- 锁的兜底释放：[../concurrency/21-lwlock](../concurrency/21-lwlock.md)、[../concurrency/22-heavyweight-lock](../concurrency/22-heavyweight-lock.md)
- 命令循环：[../process/50-postmaster](../process/50-postmaster.md)
