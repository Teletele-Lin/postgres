# 事务系统 — 状态机与子事务

> 源码：`src/backend/access/transam/xact.c`、`transam.c`、`varsup.c`
> 头文件：`src/include/access/{xact,transam}.h`

---

## 1. 职责

管理一个 backend 内事务的**生命周期**：开始、提交、回滚、保存点（savepoint）、子事务、两阶段提交的准备。它驱动两个层面的状态机，分配 XID，触发提交/回滚时的资源清理（释放锁、丢弃临时文件、运行回调、写 WAL）。

事务系统是连接"SQL 语义（BEGIN/COMMIT/SAVEPOINT）"与"底层机制（XID、CLOG、WAL、锁）"的中枢。

---

## 2. 双层状态机

`xact.c` 用**两个**状态枚举描述事务，区分"底层执行进度"与"客户端视角的事务块"。

### 2.1 TransState —— 底层状态（`xact.c:143`）

```c
typedef enum TransState {
    TRANS_DEFAULT,     // 空闲
    TRANS_START,       // 正在开始
    TRANS_INPROGRESS,  // 事务进行中
    TRANS_COMMIT,      // 提交进行中
    TRANS_ABORT,       // 回滚进行中
    TRANS_PREPARE,     // PREPARE 进行中
} TransState;
```

描述"此刻引擎在做什么"，主要用于断言与内部协调。

### 2.2 TBlockState —— 事务块状态（`xact.c:159`）

描述"客户端的事务块走到哪一步"，状态多得多，因为要覆盖隐式/显式事务、ROLLBACK 各阶段、子事务：

```c
typedef enum TBlockState {
    /* 不在事务块中 */
    TBLOCK_DEFAULT,              // 空闲
    TBLOCK_STARTED,             // 正在跑单语句事务
    /* 事务块 */
    TBLOCK_BEGIN,               // BEGIN 收到，正开块
    TBLOCK_INPROGRESS,          // 活动事务
    TBLOCK_IMPLICIT_INPROGRESS, // 隐式 BEGIN 后的活动事务
    TBLOCK_PARALLEL_INPROGRESS, // 并行 worker 内的活动事务
    TBLOCK_END,                 // COMMIT 收到
    TBLOCK_ABORT,               // 事务已失败，等 ROLLBACK
    TBLOCK_ABORT_END,           // 已失败 + ROLLBACK 收到
    TBLOCK_ABORT_PENDING,       // 活动事务 + ROLLBACK 收到
    TBLOCK_PREPARE,             // PREPARE 收到
    /* 子事务（savepoint）*/
    TBLOCK_SUBBEGIN, TBLOCK_SUBINPROGRESS, TBLOCK_SUBRELEASE,
    TBLOCK_SUBCOMMIT, TBLOCK_SUBABORT, ... TBLOCK_SUBRESTART,
} TBlockState;
```

为什么要分两层？因为同一个底层 `TRANS_INPROGRESS` 对应多种客户端语境（单语句 / 显式 BEGIN 块 / 隐式块 / 并行 worker），而 ROLLBACK 又分"已失败待回滚"和"活动但收到回滚"等微妙状态——分层让状态转移清晰可断言。

### 2.3 TransactionStateData —— 状态栈

`xact.c:195`。事务状态是一个**栈**（`parent` 指针，`xact.c:220`），支持子事务嵌套：

```c
typedef struct TransactionStateData {
    FullTransactionId fullTransactionId; // 本（子）事务 XID（可能延迟分配）
    SubTransactionId  subTransactionId;
    TransState   state;                  // 底层状态
    TBlockState  blockState;             // 块状态
    int          nestingLevel;
    MemoryContext curTransactionContext; // 本层事务内存上下文
    ...
    struct TransactionStateData *parent; // 指向父（子）事务
} TransactionStateData;
```

`CurrentTransactionState`（`xact.c:262`）始终指向栈顶。顶层是静态的 `TopTransactionStateData`（`xact.c:249`）。每开一个 savepoint/子事务就 push 一层，RELEASE/ROLLBACK TO 就 pop。

---

## 3. 核心算法

### 3.1 命令级驱动

每条 SQL 命令前后由 tcop 调用（`postgres.c`）：

```
StartTransactionCommand()   每条命令开始（必要时隐式开启事务）
  ... 执行命令 ...
CommitTransactionCommand()  每条命令结束（单语句则隐式提交；块内则保持开启）
```

`StartTransactionCommand`/`CommitTransactionCommand`（`xact.c`）是巨大的 `switch (blockState)`，把"收到这条命令时该如何转移状态"编码进去。例如 `TBLOCK_DEFAULT` + 普通命令 → 隐式 `StartTransaction()` 进 `TBLOCK_STARTED`；命令完成后单语句则直接 `CommitTransaction()` 回 `TBLOCK_DEFAULT`。显式 `BEGIN` 则停在 `TBLOCK_INPROGRESS` 等后续命令。

### 3.2 XID 的延迟分配

事务**开始时不立即分配 XID**。只有当事务**首次需要写数据**（INSERT/UPDATE/DELETE，要往元组写 `t_xmin`）时，`GetCurrentTransactionId()` 才向 `varsup.c` 的 `GetNewTransactionId()` 申请。纯只读事务永远不消耗 XID——这是抗 XID 回卷、降低 CLOG 压力的重要优化。`varsup.c` 还负责推进全局 XID 计数器、在接近回卷阈值时触发 anti-wraparound VACUUM 警告/强制（见 [31-clog-slru](31-clog-slru.md)）。

### 3.3 提交流程（CommitTransaction）

`CommitTransaction()` 大致顺序：
1. 运行 BEFORE COMMIT 回调、AFTER 触发器（`ExecutorFinish` 已部分处理）。
2. 处理待删文件、待发的缓存失效消息。
3. **写 commit WAL 记录**（`xl_xact_commit`，含失效消息、待删关系、子事务 XID）。
4. **在 CLOG 标记本 XID 为 COMMITTED**（`TransactionIdCommitTree`，见 [31](31-clog-slru.md)）—— 这是提交的"原子时刻"：CLOG 一旦标记，对其他事务即生效。
5. 若 `synchronous_commit`，等 WAL flush（及同步备库确认，见 [../replication/60-physical](../replication/60-physical.md)）。
6. 释放锁、广播失效消息、清理事务内存上下文。

注意 **WAL 先写、CLOG 后标**：崩溃恢复靠 WAL 重建 CLOG 状态。

### 3.4 回滚（AbortTransaction）

`ereport(ERROR)` 经 `siglongjmp` 跳到事务边界，`AbortCurrentTransaction()` → `AbortTransaction()`：在 CLOG 标 ABORTED（或干脆不标——未标记的 XID 默认视为 abort）、释放锁、回滚缓存失效、丢弃本事务的临时文件，**重置事务内存上下文一次性回收所有事务内分配的内存**（见 [../infra/70-memory-context](../infra/70-memory-context.md)）。

### 3.5 子事务（savepoint）

`DefineSavepoint`/`ReleaseSavepoint`/`RollbackToSavepoint` 操作状态栈：
- `SAVEPOINT s` → push 一个子事务（分配 subxid，记入 subtrans SLRU 映射子→父）。
- `ROLLBACK TO s` → abort 到该层，丢弃其后的子事务效果，但保留外层。
- 子事务的存在让单条语句出错不必回滚整个事务块（PL/pgSQL 的 `EXCEPTION` 块就是用子事务实现的）。

---

## 4. 设计模式

- **双层状态机分离关注点**：底层 `TransState`（引擎进度）与 `TBlockState`（客户端事务块）解耦，让繁多的事务块语境与回滚阶段各自清晰可断言。
- **状态栈支持嵌套**：用 `parent` 链的栈表达子事务/savepoint 的嵌套，push/pop 对应 SAVEPOINT/RELEASE。
- **命令级驱动（Start/Commit per command）**：把"每条命令的状态转移"集中在两个大 switch，tcop 无脑调用，事务语义全在 xact.c。
- **XID 延迟分配**：只读不耗 XID，把稀缺的 32 位 XID 资源留给真正的写事务，缓解回卷。
- **提交的原子时刻（CLOG 标记）**：提交是否"生效"由 CLOG 那一位决定，且必在 WAL 持久化之后，保证崩溃可恢复。
- **回滚即上下文重置**：异常清理依赖 MemoryContext 的区域回收，而非逐对象 free。

---

## 5. 架构编排

```
tcop（postgres.c，每条命令）
  ├─ StartTransactionCommand   switch(blockState)  xact.c
  │     └─ StartTransaction（建 TransactionState、事务内存上下文；XID 延迟）
  ├─ 执行命令（首次写数据 → GetCurrentTransactionId → varsup.c 分配 XID）
  └─ CommitTransactionCommand  switch(blockState)
        ├─ 单语句/COMMIT → CommitTransaction
        │     → 写 xl_xact_commit（WAL）→ CLOG 标 COMMITTED → 同步等待 → 放锁/发失效
        ├─ BEGIN 块 → 保持 TBLOCK_INPROGRESS
        └─ ERROR → AbortTransaction（CLOG 标 ABORTED、放锁、重置上下文）
子事务：DefineSavepoint/RollbackToSavepoint → push/pop 状态栈 + subtrans SLRU
```

---

## 6. 动手探索

```sql
-- XID 延迟分配：只读事务无 XID
BEGIN; SELECT txid_current_if_assigned();   -- NULL（还没写）
       INSERT INTO t VALUES(1);
       SELECT txid_current_if_assigned();   -- 现在有了
COMMIT;

-- 子事务 / savepoint
BEGIN;
  INSERT INTO t VALUES(1);
  SAVEPOINT s1;
  INSERT INTO t VALUES(2);
  ROLLBACK TO s1;        -- 撤销 2，保留 1
COMMIT;

-- 观察事务状态
SELECT pid, state, backend_xid, backend_xmin FROM pg_stat_activity;
```

---

## 相关模块

- XID 状态存储：[31-clog-slru](31-clog-slru.md)
- 可见性：[32-mvcc-snapshot](32-mvcc-snapshot.md)
- 持久化：[33-wal](33-wal.md)
- 两阶段提交：[35-twophase](35-twophase.md)
- 内存清理：[../infra/70-memory-context](../infra/70-memory-context.md)
