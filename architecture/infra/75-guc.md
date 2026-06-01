# GUC — 配置参数系统

> 源码：`src/backend/utils/misc/{guc,guc_tables,guc_funcs}.c`、`src/backend/utils/misc/guc-file.l`
> 头文件：`src/include/utils/guc.h`、`guc_tables.h`

---

## 1. 职责

GUC（**G**rand **U**nified **C**onfiguration）是 PostgreSQL 的统一配置系统：管理所有可调参数（`shared_buffers`、`work_mem`、`search_path`、`enable_hashjoin`…）。它负责参数的定义、来源优先级、生效时机、类型校验、以及 `SET`/`SHOW`/`postgresql.conf`/命令行的统一处理。扩展也通过它注册自定义参数。

---

## 2. 参数的上下文（生效时机）

`src/include/utils/guc.h:74` 的 `GucContext` 决定一个参数**何时能改、改了何时生效**，从严到松：

| 上下文 | 何时可改 / 生效 | 例 |
|--------|----------------|-----|
| `PGC_INTERNAL` | 只读，内部固定 | `block_size`、`server_version` |
| `PGC_POSTMASTER` | 只能启动时设，**改需重启** | `shared_buffers`、`max_connections`、`wal_level` |
| `PGC_SIGHUP` | 改 `postgresql.conf` + reload（SIGHUP）生效 | `autovacuum`、`log_*`、`checkpoint_timeout` |
| `PGC_SUSET` | 超级用户可在会话内 `SET` | `session_preload_libraries` 类 |
| `PGC_BACKEND` / `PGC_SU_BACKEND` | 连接建立时固定 | `log_connections` |
| `PGC_USERSET` | 任何用户随时 `SET` | `work_mem`、`search_path`、`enable_*` |

这把"改这个参数有多大影响/多危险"编码进上下文：影响共享内存布局的必须重启（`PGC_POSTMASTER`），纯会话偏好可随意改（`PGC_USERSET`）。

---

## 3. 参数来源与优先级

同一参数可能从多处设置，`GucSource`（`guc.h`）定义优先级，**高优先级覆盖低优先级**（从低到高）：

```
默认值（编译内置）
 < postgresql.conf（PGC_S_FILE）
 < postgresql.auto.conf（ALTER SYSTEM 写的，PGC_S_FILE）
 < 命令行 / 环境变量
 < ALTER DATABASE / ALTER ROLE（登录时按库/角色应用，PGC_S_DATABASE/ROLE）
 < 会话 SET（PGC_S_SESSION）
 < SET LOCAL（仅当前事务，最高）
```

`SHOW x` 显示当前生效值，`pg_settings` 视图能看到来源（`source`）、上下文、边界等全部元信息。`RESET x` 回退到 SET 之前的值。

---

## 4. 核心数据结构

`guc_tables.c` 用按类型分的静态数组定义所有内置参数。每个参数一个 `config_generic` 派生结构（bool/int/real/string/enum）：

```c
struct config_int {
    struct config_generic gen;   // name, context, group, short_desc, flags...
    int        *variable;        // 指向实际存值的 C 变量
    int         boot_val;        // 默认值
    int         min, max;        // 范围
    GucIntCheckHook  check_hook; // 赋值前校验
    GucIntAssignHook assign_hook;// 赋值后副作用
    GucShowHook      show_hook;
};
```

- `variable` 指向真正被代码读取的全局变量（如 `work_mem`），GUC 改它一处，全代码自动看到新值。
- **check_hook**：赋值前校验/规范化（如 `search_path` 解析、单位换算）。
- **assign_hook**：赋值后触发副作用（如改 `log_min_messages` 更新日志阈值）。

参数还带 **flags**：`GUC_UNIT_KB/MS/...`（单位，`'4MB'`→数值）、`GUC_LIST_INPUT`、`GUC_NO_SHOW_ALL`、`GUC_REPORT`（值变了主动告知客户端，如 `client_encoding`）等。

---

## 5. 核心机制

### 5.1 设置流程

`SET work_mem = '64MB'`：解析 → 查 GUC 表 → 单位换算（`64MB`→65536 kB）→ check_hook 校验范围 → 写 `*variable` + 记入栈 → assign_hook 副作用。

### 5.2 事务性与栈

`SET`（非 LOCAL）在会话内持久，但若在事务中 `SET` 后回滚，GUC 会**随事务回滚**——GUC 维护一个变更栈，`AtEOXact_GUC` 在提交/回滚时确认或撤销本事务的 SET。`SET LOCAL` 只在当前事务有效，事务结束即还原。这让配置变更也具备事务语义。

### 5.3 reload

`pg_ctl reload`/`SELECT pg_reload_conf()` 发 SIGHUP，各进程重读 `postgresql.conf`，对 `PGC_SIGHUP` 及以下上下文的参数应用新值（`PGC_POSTMASTER` 的改动被忽略并告警，需重启）。

---

## 6. 扩展自定义参数

`guc.h:358` 起的 `DefineCustomXxxVariable`：

```c
DefineCustomIntVariable("my_ext.threshold",
    "描述", NULL, &my_threshold,
    100, 0, INT_MAX, PGC_USERSET, 0, NULL, NULL, NULL);
```

扩展在 `_PG_init` 里注册带 **前缀**（`my_ext.`）的参数，之后就能像内置参数一样 `SET my_ext.threshold = ...`、出现在 `pg_settings`。这让扩展无需自造配置机制。

---

## 7. 设计模式

- **上下文编码生效时机**：用 `GucContext` 把"改这个参数多危险/何时生效"声明式地绑到参数上，统一决定重启/reload/会话级，避免散落的特判。
- **来源优先级（分层覆盖）**：默认 < 配置文件 < 库/角色 < 会话 < 事务，用清晰的优先级链让多来源配置可预测地合成。
- **变量指针 + 钩子**：参数直连一个 C 全局变量（一处改、处处见），check/assign hook 注入校验与副作用，把"配置"与"使用配置的代码"解耦。
- **事务性配置（GUC 栈）**：让 `SET`/`SET LOCAL` 随事务提交/回滚，配置变更具备与数据一致的事务语义。
- **可扩展注册（DefineCustom*）**：扩展复用同一套配置基础设施注册带前缀的参数，无需另造轮子。
- **单位与 flags 声明式处理**：用 flags 表达单位换算、列表输入、变更上报等横切行为，集中而非分散实现。

---

## 8. 架构编排

```
定义：guc_tables.c（内置）/ DefineCustom*（扩展，_PG_init）
  → config_generic{name,context,flags} + *variable + boot/min/max + check/assign hook

设置来源（低→高覆盖）：
  默认 → postgresql.conf → postgresql.auto.conf(ALTER SYSTEM)
       → ALTER DATABASE/ROLE → 会话 SET → SET LOCAL
  每次赋值：单位换算 → check_hook → 写 *variable → assign_hook

生效：
  PGC_POSTMASTER：仅启动；PGC_SIGHUP：reload(SIGHUP)；PGC_USERSET：随时 SET
  事务回滚 → AtEOXact_GUC 撤销本事务的 SET

读取：代码直接读 *variable（如 work_mem）；SHOW/pg_settings 展示
```

---

## 9. 动手探索

```sql
-- 全部参数的元信息：上下文、来源、边界、单位、是否需重启
SELECT name, setting, unit, context, source, boot_val, min_val, max_val, pending_restart
FROM pg_settings WHERE name IN ('shared_buffers','work_mem','enable_hashjoin');

-- 各种设置方式
SET work_mem = '64MB';            -- 会话级
SET LOCAL work_mem = '8MB';       -- 仅当前事务
ALTER SYSTEM SET work_mem = '32MB';  -- 写 postgresql.auto.conf
SELECT pg_reload_conf();          -- reload（对 SIGHUP 类生效）
ALTER DATABASE db SET search_path = a,b;  -- 库级默认
RESET work_mem;  SHOW work_mem;

-- 哪些改了需重启
SELECT name FROM pg_settings WHERE pending_restart;
```

---

## 相关模块

- 优化器代价参数：[../query/05-planner-paths-joins](../query/05-planner-paths-joins.md)
- 内存相关（work_mem/shared_buffers）：[70-memory-context](70-memory-context.md)、[../storage/16-buffer-manager](../storage/16-buffer-manager.md)
- 配置文件加载（postmaster）：[../process/50-postmaster](../process/50-postmaster.md)
- 扩展注册时机：扩展的 `_PG_init`
