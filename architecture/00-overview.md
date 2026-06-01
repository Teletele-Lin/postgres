# PostgreSQL 整体架构总览

> 适用版本：本仓库 `master` 分支（PostgreSQL **19devel**）。

本文建立全局心智模型：进程模型 → 共享内存布局 → 一条 SQL 的完整处理流水线 → 存储栈分层。
各子系统的细节见对应模块文档。

---

## 1. 设计哲学：多进程 + 共享内存

PostgreSQL 采用 **多进程架构**（每连接一个 backend 进程），而非多线程。这是 1980 年代 POSTGRES 项目延续至今的根本决策，理解它是理解整个内核的前提：

- **隔离性**：一个 backend 崩溃（段错误）不会直接踩坏其他 backend 的私有内存。Postmaster 检测到子进程异常退出后，会让所有 backend 重启以重置共享内存（见 [`process/50-postmaster.md`](process/50-postmaster.md)）。
- **共享状态集中在共享内存**：Buffer Pool、WAL Buffers、锁表、PGPROC 数组、CLOG 缓冲等都放在 `mmap`/`shmget` 出来的共享内存段中，由各种锁（spinlock / LWLock / 重量级锁）保护。
- **私有状态在进程本地**：MemoryContext 体系、RelCache、CatCache、当前事务状态、各种 backend-local 缓存都是每进程一份。
- **代价**：进程间通信（IPC）比线程间昂贵；并行查询需要借助动态共享内存（DSM）和共享消息队列在 worker 间传递数据（见 [`process/53-parallel-query.md`](process/53-parallel-query.md)）。

### 进程家族

```
postmaster（主守护进程，不接触共享内存数据页）
├── backend            每个客户端连接 fork 一个，跑 PostgresMain() 主循环
├── background workers  并行 worker、逻辑复制 worker、扩展自定义 worker
├── checkpointer       周期性写检查点，刷脏页
├── background writer   后台预刷脏页，平滑 I/O
├── walwriter          周期性 fsync WAL
├── autovacuum launcher → autovacuum worker（按需 fork）
├── archiver           执行 archive_command 归档 WAL 段
├── walsender          每个流复制 / 逻辑复制连接一个，发送 WAL
├── walreceiver        备库上接收主库 WAL（仅 standby）
├── startup            崩溃恢复 / 备库回放 WAL（仅恢复期间）
├── logical replication launcher / apply worker
└── io workers         AIO 子系统的 I/O 工作进程（worker 模式）
```

详见 [`process/51-background-procs.md`](process/51-background-procs.md)。

---

## 2. 模块全景图

| 子系统 | 源码目录 | 职责 | 模块文档 |
|------|----------|------|---------|
| Parser | `src/backend/parser/` | SQL 字符串 → RawStmt 语法树 | [query/01](query/01-parser.md) |
| Analyzer | `src/backend/parser/` | RawStmt → Query 查询树 | [query/02](query/02-analyzer.md) |
| Rewriter | `src/backend/rewrite/` | 视图展开、规则系统、RLS | [query/03](query/03-rewriter.md) |
| Planner | `src/backend/optimizer/` | Query → PlannedStmt | [query/04](query/04-planner-overview.md), [05](query/05-planner-paths-joins.md) |
| Executor | `src/backend/executor/` | Plan 树的火山模型执行 | [query/06](query/06-executor-overview.md)–[08](query/08-executor-nodes.md) |
| Table AM | `src/backend/access/table,heap/` | 表存储抽象 + Heap 实现 | [storage/10](storage/10-table-am.md), [11](storage/11-heap.md) |
| Index AM | `src/backend/access/{nbtree,gin,...}/` | 索引存储抽象 + 各索引实现 | [storage/12](storage/12-index-am.md)–[14](storage/14-other-indexes.md) |
| Storage | `src/backend/storage/` | Buffer、Page、SMGR、文件 I/O | [storage/15](storage/15-page-layout.md)–[17](storage/17-smgr-forks.md) |
| Lock Manager | `src/backend/storage/lmgr/` | spinlock/LWLock/重量级锁/谓词锁 | [concurrency/20](concurrency/20-lock-overview.md)–[23](concurrency/23-predicate-ssi.md) |
| Transaction | `src/backend/access/transam/` | XID、CLOG、状态机、2PC | [transaction/30](transaction/30-xact.md), [31](transaction/31-clog-slru.md), [35](transaction/35-twophase.md) |
| MVCC | `heapam_visibility.c` + `snapmgr.c` | 快照与可见性判断 | [transaction/32](transaction/32-mvcc-snapshot.md) |
| WAL | `xlog*.c` | 预写日志、检查点、恢复 | [transaction/33](transaction/33-wal.md), [34](transaction/34-recovery-checkpoint.md) |
| Catalog | `src/backend/catalog/` | 系统表定义与访问 | [catalog/40](catalog/40-catalog.md) |
| Caches | `src/backend/utils/cache/` | RelCache / SysCache / 失效 | [catalog/41](catalog/41-caches.md) |
| Postmaster | `src/backend/postmaster/` | 进程管理与后台进程 | [process/50](process/50-postmaster.md), [51](process/51-background-procs.md) |
| Replication | `src/backend/replication/` | 物理 / 逻辑复制 | [replication/60](replication/60-physical.md), [61](replication/61-logical.md) |
| Commands | `src/backend/commands/` | CREATE/ALTER/VACUUM/COPY 等 DDL/DML | 散见各模块 |
| Utilities | `src/backend/utils/` | MemoryContext、elog、GUC、Node | [infra/70](infra/70-memory-context.md)–[75](infra/75-guc.md) |

---

## 3. 一条 SQL 的完整旅程

### 3.1 顶层流水线

简单查询协议（simple query protocol）的主入口是 `exec_simple_query()`
（`src/backend/tcop/postgres.c:1029`），它串起整条流水线：

```
SQL 字符串
  │  pg_parse_query()                    src/backend/tcop/postgres.c:616
  ▼
RawStmt 列表（语法树，未查库）
  │  pg_analyze_and_rewrite_*()          postgres.c:682
  │    ├─ parse_analyze_*()  语义分析     → Query
  │    └─ QueryRewrite()     重写         → Query 列表
  ▼
Query 列表（查询树，名称/类型已解析）
  │  pg_plan_queries()                   postgres.c:987
  ▼
PlannedStmt 列表（物理执行计划）
  │  PortalRun() → ExecutorStart/Run/Finish/End
  ▼
结果元组 → DestReceiver → libpq 协议 → 客户端
```

各阶段对应的函数与详细行为：

| 阶段 | 入口函数 | 输入 → 输出 | 关键点 |
|------|---------|------------|--------|
| 解析 | `pg_parse_query` | `char *` → `List<RawStmt>` | **不访问数据库**，纯语法 |
| 分析 | `parse_analyze_*` | `RawStmt` → `Query` | 打开表、解析列名、类型推导 |
| 重写 | `QueryRewrite` | `Query` → `List<Query>` | 视图展开、规则、RLS |
| 规划 | `pg_plan_queries` | `Query` → `PlannedStmt` | 路径枚举、代价估算 |
| 执行 | `ExecutorRun` | `PlannedStmt` → 元组 | 火山模型拉取 |

### 3.2 为什么分这么多阶段？

- **解析与分析分离**：`gram.y` 的语法规则**不允许访问数据库**（`src/backend/parser/gram.y` 开头注释）。因为一个多语句字符串（如 `CREATE TABLE t; INSERT INTO t ...`）会在执行第一条之前被整体解析完，此时 `t` 还不存在。所有需要查目录的工作（表/列/类型/函数解析）推迟到 parse analysis。
- **分析与规划分离**：`Query` 是逻辑表示（"要什么"），`PlannedStmt` 是物理表示（"怎么做"）。同一个 `Query` 可以生成多种执行计划，由优化器按代价择优。
- **缓存边界**：扩展查询协议（extended protocol，`PREPARE`/`Bind`/`Execute`）正是利用这些边界，缓存 parse tree 和 plan，避免重复解析与规划。

### 3.3 扩展查询协议

`PostgresMain()`（`src/backend/tcop/postgres.c:4274`）的主循环按消息类型分发：

- `'Q'`（Query）→ `exec_simple_query()`：一次走完全流程。
- `'P'`（Parse）→ `exec_parse_message()`：解析 + 分析，生成命名 prepared statement，缓存 `CachedPlanSource`。
- `'B'`（Bind）→ `exec_bind_message()`：绑定参数，生成 portal，可能复用缓存计划（generic plan vs custom plan 的抉择见 `src/backend/utils/cache/plancache.c`）。
- `'E'`（Execute）→ `exec_execute_message()`：执行 portal，可分批返回。

---

## 4. 存储栈分层

从一条 SQL 看到一行数据，到这行数据落在磁盘上，要穿过这些层：

```
执行节点（如 SeqScan）        query/08
  │ table_getnextslot()
  ▼
Table AM（TableAmRoutine 虚表）  storage/10
  │ heap_getnextslot()
  ▼
Heap 访问方法                  storage/11
  │ ReadBuffer()
  ▼
Buffer Manager（缓冲池 + 时钟扫描 + 哈希表）  storage/16
  │ smgrread() / smgrwrite()
  ▼
SMGR（存储管理器抽象） → md.c   storage/17
  │ pread() / pwrite() / AIO
  ▼
操作系统文件（每 relation = 多个 1GB 段文件 × 多个 fork）
```

并行地，所有修改先经 **WAL**（预写日志）保证持久性与可恢复性：

```
heap_insert/update/delete 等
  │ XLogInsert()                transaction/33
  ▼
WAL Buffer（共享内存环形缓冲）
  │ XLogFlush() → fsync
  ▼
pg_wal/ 下的 WAL 段文件
  │ walsender 流式发送
  ▼
备库 / 归档
```

**WAL 先行规则（Write-Ahead Logging）**：任何对数据页的修改，其对应的 WAL 记录必须先于脏页落盘。
这由 buffer manager 在刷页前检查页的 LSN 与 WAL flush 位置来保证（见 [`storage/16`](storage/16-buffer-manager.md) 与 [`transaction/33`](transaction/33-wal.md)）。

---

## 5. 并发控制全景

PostgreSQL 用 **MVCC（多版本并发控制）** 让读写互不阻塞：

- **写不阻塞读**：`UPDATE` 不原地改，而是插入新版本元组、把旧版本的 `t_xmax` 标记为当前 XID。旧读者仍看旧版本。
- **可见性**：每条元组带 `t_xmin`（创建它的事务）和 `t_xmax`（删除它的事务）。`HeapTupleSatisfiesMVCC()` 结合快照判断元组对当前事务是否可见（见 [`transaction/32`](transaction/32-mvcc-snapshot.md)）。
- **快照**：`GetSnapshotData()` 扫描共享内存 ProcArray，记录"此刻正在运行的 XID 集合"。
- **清理**：旧版本由 `VACUUM` 与 HOT prune 回收（见 [`storage/11`](storage/11-heap.md)）。

锁体系是四层的（从轻到重，见 [`concurrency/20`](concurrency/20-lock-overview.md)）：

| 层 | 用途 | 粒度 | 死锁检测 |
|----|------|------|---------|
| Spinlock | 极短临界区（LWLock 内部状态） | 纳秒级 | 无 |
| LWLock | 共享内存结构的读写保护（buffer、WAL 等） | 微秒级 | 无 |
| 重量级锁 | SQL 对象锁（表锁、行锁） | 语句/事务级 | 有 |
| 谓词锁 | Serializable（SSI）冲突追踪 | 事务级 | SSI 算法 |

---

## 6. 关键全局数据结构速查

| 结构 | 定义位置 | 含义 | 模块 |
|------|---------|------|------|
| `Query` | `src/include/nodes/parsenodes.h` | 分析后的查询树 | query/02 |
| `PlannedStmt` | `src/include/nodes/plannodes.h` | 物理执行计划 | query/04 |
| `RelOptInfo` / `Path` | `src/include/nodes/pathnodes.h` | 优化器的关系与路径 | query/05 |
| `PlanState` / `EState` | `src/include/nodes/execnodes.h` | 执行期状态树 | query/06 |
| `TupleTableSlot` | `src/include/executor/tuptable.h` | 元组容器 | query/06 |
| `HeapTupleHeaderData` | `src/include/access/htup_details.h` | 元组物理头 | storage/15 |
| `BufferDesc` | `src/include/storage/buf_internals.h` | 缓冲区描述符 | storage/16 |
| `PGPROC` | `src/include/storage/proc.h` | 每 backend 的共享内存结构 | concurrency/22 |
| `SnapshotData` | `src/include/utils/snapshot.h` | MVCC 快照 | transaction/32 |
| `XLogRecord` | `src/include/access/xlogrecord.h` | WAL 记录头 | transaction/33 |
| `RelationData` | `src/include/utils/rel.h` | 打开的关系描述符 | catalog/41 |

---

## 7. 贯穿全局的设计模式

理解这几个反复出现的模式，能让阅读任何子系统都事半功倍：

1. **虚表 / 策略模式（Routine struct of function pointers）**：`TableAmRoutine`、`IndexAmRoutine`、`TupleTableSlotOps`、`MemoryContextMethods`、`DestReceiver`、各 RMGR 的 `rm_redo`。PostgreSQL 用"函数指针结构体 + 注册表"实现 C 语言里的多态。
2. **资源管理器分发（RMGR）**：WAL 回放、逻辑解码都通过 `RmgrTable[xl_rmid]` 分发到对应资源管理器（见 [`transaction/33`](transaction/33-wal.md)）。
3. **基于区域的内存管理（MemoryContext）**：不手工 `free`，而是 `MemoryContextReset/Delete` 批量释放。executor 每处理一行就 reset per-tuple context（见 [`infra/70`](infra/70-memory-context.md)）。
4. **节点系统（Node）**：所有树结构第一个字段是 `NodeTag`，配合 `gen_node_support.pl` 自动生成 copy/equal/out/read 函数（见 [`infra/72`](infra/72-node-system.md)）。
5. **缓存 + 失效消息（cache invalidation）**：RelCache/CatCache 在本地缓存系统表，通过共享 SI 消息队列在事务提交时广播失效（见 [`catalog/41`](catalog/41-caches.md)）。
6. **异常即 longjmp**：`ereport(ERROR)` 通过 `siglongjmp` 跳回最近的 `PG_TRY/PG_CATCH` 或事务边界，配合 MemoryContext 自动清理（见 [`infra/71`](infra/71-error-elog.md)）。
7. **火山模型（Volcano iterator）**：执行器每个节点暴露统一的 `ExecProcNode()`，父节点按需向子节点拉取一行（见 [`query/06`](query/06-executor-overview.md)）。

---

## 8. 如何动手探索

```bash
# 构建（Meson）
meson setup build --prefix=$PGHOME -Dcassert=true -Dbuildtype=debugoptimized
ninja -C build install

# 看某条 SQL 的计划
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;

# 看 WAL 记录
pg_waldump $PGDATA/pg_wal/000000010000000000000001

# 看页面内部（contrib 扩展）
CREATE EXTENSION pageinspect;
SELECT * FROM heap_page_items(get_raw_page('t', 0));

# 用 gdb 跟踪某个 backend
SELECT pg_backend_pid();   -- 拿到 pid 后 gdb -p <pid>
```

每个模块文档末尾都会给出对应的"动手探索"建议。
