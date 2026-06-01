# PostgreSQL 内核架构文档

> 目标：以"职责 / 核心数据结构 / 核心算法 / 设计模式 / 架构编排"五个维度，深度结合源码注释与实现，
> 帮助读者系统地理解 PostgreSQL 内核。每个模块一个独立文件，代码引用均标注 `文件:行号`。


## 阅读顺序建议

1. 先读 [`00-overview.md`](00-overview.md) 建立全局心智模型（进程模型 + 查询处理流水线 + 存储栈）。
2. 想理解"一条 SQL 如何被执行" → 读 `query/` 全套。
3. 想理解"数据如何落盘、如何并发读写" → 读 `storage/` 与 `concurrency/`。
4. 想理解"崩溃如何恢复、可见性如何保证" → 读 `transaction/`。
5. `catalog/` `process/` `replication/` `infra/` 为支撑性子系统，按需查阅。

## 目录结构

### 总览
- [`00-overview.md`](00-overview.md) — 整体架构、进程模型、查询处理流水线总览

### query/ — 查询处理流水线
- [`query/01-parser.md`](query/01-parser.md) — 词法与语法解析（flex/bison → RawStmt）
- [`query/02-analyzer.md`](query/02-analyzer.md) — 语义分析（RawStmt → Query）
- [`query/03-rewriter.md`](query/03-rewriter.md) — 重写器（视图展开 / 规则 / RLS）
- [`query/04-planner-overview.md`](query/04-planner-overview.md) — 优化器总览与编排
- [`query/05-planner-paths-joins.md`](query/05-planner-paths-joins.md) — 路径、代价估算与 join 枚举
- [`query/06-executor-overview.md`](query/06-executor-overview.md) — 执行器总览与火山模型
- [`query/07-executor-expr.md`](query/07-executor-expr.md) — 表达式编译与解释执行
- [`query/08-executor-nodes.md`](query/08-executor-nodes.md) — 各执行节点（扫描 / join / 聚合 / 排序）

### storage/ — 存储引擎
- [`storage/10-table-am.md`](storage/10-table-am.md) — 表访问方法抽象层
- [`storage/11-heap.md`](storage/11-heap.md) — Heap 表实现与 HOT
- [`storage/12-index-am.md`](storage/12-index-am.md) — 索引访问方法抽象层
- [`storage/13-nbtree.md`](storage/13-nbtree.md) — B-tree（Lehman-Yao）
- [`storage/14-other-indexes.md`](storage/14-other-indexes.md) — GIN / GiST / BRIN / Hash / SP-GiST
- [`storage/15-page-layout.md`](storage/15-page-layout.md) — 页面布局与元组格式
- [`storage/16-buffer-manager.md`](storage/16-buffer-manager.md) — 缓冲区管理与时钟扫描
- [`storage/17-smgr-forks.md`](storage/17-smgr-forks.md) — SMGR / fork / FSM / VM / AIO

### concurrency/ — 并发控制与锁
- [`concurrency/20-lock-overview.md`](concurrency/20-lock-overview.md) — 四层锁体系总览
- [`concurrency/21-lwlock.md`](concurrency/21-lwlock.md) — Spinlock / LWLock / Latch
- [`concurrency/22-heavyweight-lock.md`](concurrency/22-heavyweight-lock.md) — 重量级锁与死锁检测
- [`concurrency/23-predicate-ssi.md`](concurrency/23-predicate-ssi.md) — 谓词锁与 SSI

### transaction/ — 事务、MVCC 与 WAL
- [`transaction/30-xact.md`](transaction/30-xact.md) — 事务状态机与子事务
- [`transaction/31-clog-slru.md`](transaction/31-clog-slru.md) — CLOG / SLRU / subtrans / multixact
- [`transaction/32-mvcc-snapshot.md`](transaction/32-mvcc-snapshot.md) — MVCC 快照与可见性
- [`transaction/33-wal.md`](transaction/33-wal.md) — WAL 写入与记录格式
- [`transaction/34-recovery-checkpoint.md`](transaction/34-recovery-checkpoint.md) — 检查点与崩溃恢复
- [`transaction/35-twophase.md`](transaction/35-twophase.md) — 两阶段提交

### catalog/ — 系统目录与缓存
- [`catalog/40-catalog.md`](catalog/40-catalog.md) — 系统表 / BKI / bootstrap
- [`catalog/41-caches.md`](catalog/41-caches.md) — RelCache / SysCache / 失效消息

### process/ — 进程模型与 IPC
- [`process/50-postmaster.md`](process/50-postmaster.md) — Postmaster 与进程生命周期
- [`process/51-background-procs.md`](process/51-background-procs.md) — bgwriter / checkpointer / autovacuum 等
- [`process/52-shmem-ipc.md`](process/52-shmem-ipc.md) — 共享内存与 IPC
- [`process/53-parallel-query.md`](process/53-parallel-query.md) — 并行查询基础设施

### replication/ — 复制
- [`replication/60-physical.md`](replication/60-physical.md) — 物理流复制
- [`replication/61-logical.md`](replication/61-logical.md) — 逻辑解码与发布订阅

### infra/ — 基础设施
- [`infra/70-memory-context.md`](infra/70-memory-context.md) — 内存上下文
- [`infra/71-error-elog.md`](infra/71-error-elog.md) — 错误处理与 elog
- [`infra/72-node-system.md`](infra/72-node-system.md) — 节点系统
- [`infra/73-collections.md`](infra/73-collections.md) — List / Bitmapset / HTAB
- [`infra/74-datum-types.md`](infra/74-datum-types.md) — Datum 与类型系统
- [`infra/75-guc.md`](infra/75-guc.md) — GUC 配置系统

## 文档约定

- **代码引用**：形如 `src/backend/access/heap/heapam.c:1234`，可在编辑器中点击跳转。行号基于本仓库当前 `master` 分支，重构后可能漂移，以函数名为准。
- **代码片段**：保留 C 原文以确保精确，注释翻译为中文并以 `//` 或 `/* */` 标注。
- **`[[链接]]`** 风格的交叉引用指向本目录其他文件的相对路径。
