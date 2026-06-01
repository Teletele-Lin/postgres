# 两阶段提交（2PC）

> 源码：`src/backend/access/transam/twophase.c`
> 相关：[30-xact](30-xact.md)、[33-wal](33-wal.md)、[34-recovery-checkpoint](34-recovery-checkpoint.md)

---

## 1. 职责

支持 **分布式事务**所需的两阶段提交协议：把"提交"拆成 **PREPARE**（准备好、保证一定能提交）和 **COMMIT PREPARED / ROLLBACK PREPARED**（真正提交/回滚）两步，中间事务可以跨连接、跨重启存活。外部事务管理器（XA / 应用层协调者）借此让多个数据库原子地一起提交或一起回滚。

SQL 接口：
```sql
PREPARE TRANSACTION 'gid';        -- 第一阶段：准备，gid 是全局事务标识符
COMMIT PREPARED 'gid';            -- 第二阶段：提交
ROLLBACK PREPARED 'gid';          -- 第二阶段：回滚
```

---

## 2. 与普通提交的关键差异

普通事务的状态（锁、XID、待发失效消息）都在 **backend 进程的内存里**，连接断开即消失。而 prepared 事务必须：
- **脱离原 backend 存活**：PREPARE 后原连接可断开，任何连接都能 `COMMIT PREPARED`。
- **跨崩溃存活**：PREPARE 后即使服务器重启，事务仍处于"已准备、待裁决"状态。
- **持有的锁继续保持**：直到第二阶段才释放。

因此 2PC 的核心是把"事务的全部待提交状态"**序列化到持久存储**，并能在任意进程/重启后重建。

---

## 3. 核心数据结构

### 3.1 GlobalTransaction（共享内存）

PREPARE 时，事务的状态从 backend 内存"移交"到共享内存的 `GlobalTransaction` 槽（`twophase.c`）：含 `gid`（全局事务标识，`twophase.c:172`，≤ `GIDSIZE`）、`FullTransactionId`、owner、prepared_at 时间、以及一个"虚拟" PGPROC（让该事务能在 ProcArray 中继续持有 XID 和锁，即便没有真实 backend）。

### 3.2 状态文件（pg_twophase/）

prepared 事务的完整状态序列化为 `pg_twophase/<xid>` 文件（`TWOPHASE_DIR`，`twophase.c:115`），含 `TwoPhaseFileHeader` + 各子记录：
- 涉及的子事务 XID。
- **持有的锁**（`AccessExclusiveLock` 等，恢复时要重新获取）。
- 待提交的缓存失效消息。
- 待删除/待创建的关系文件（commit/abort 时分别处理）。
- 通知（NOTIFY）。

`twophase.c:34,44,54` 的注释描述了它的生命周期：每个 prepared 事务一个状态文件；检查点时把状态写入文件；恢复开始时扫描 `pg_twophase` 一次重建所有 prepared 事务。

---

## 4. 核心算法

### 4.1 第一阶段：PrepareTransaction

`PrepareTransaction()`（`twophase.c`）：

```
1. MarkAsPreparing：在共享内存分配 GlobalTransaction 槽，绑定 gid + XID   twophase.c:365
2. 收集本事务全部待提交状态（锁、失效消息、待删文件、子事务…）
3. 序列化为 pg_twophase 状态文件内容，写 PREPARE WAL 记录（持久化关键点）
4. 在 CLOG 中本 XID 仍保持"进行中"（既未 commit 也未 abort）
5. 把锁等状态从 backend 转交给 GlobalTransaction（原 backend 不再持有）
6. 原 backend 事务结束，但 XID 与锁由 GlobalTransaction 继续持有
```

PREPARE 一旦成功（WAL 落盘），该事务就**保证可提交**——所有约束已检查、所有需要的资源已锁定。这是 2PC 协议对协调者的承诺。

### 4.2 第二阶段：COMMIT/ROLLBACK PREPARED

任意 backend 执行 `COMMIT PREPARED 'gid'`：
```
1. 按 gid 找到 GlobalTransaction
2. 读回状态文件（或仍在内存）
3. 写 COMMIT PREPARED WAL 记录 + 在 CLOG 标 COMMITTED（提交的原子时刻）
4. 执行待提交副作用：发失效消息、删除应删文件、释放锁
5. 删除 pg_twophase 状态文件，释放 GlobalTransaction 槽
```
`ROLLBACK PREPARED` 类似，但 CLOG 标 ABORTED、执行 abort 时的副作用。

### 4.3 崩溃恢复中的重建

恢复结束时 `RecoverPreparedTransactions()`（`twophase.c:54` 注释）扫描 `pg_twophase` 目录（以及 WAL 中尚未落到文件的 PREPARE 记录），为每个 prepared 事务**重建 GlobalTransaction、重新获取它持有的锁**。这样重启后这些事务仍"悬而未决"，等待协调者裁决。这也是为什么 [34-recovery-checkpoint](34-recovery-checkpoint.md) 的收尾要调它。

### 4.4 状态文件与 WAL 的协同

为减少 I/O，近期 prepared 事务的状态优先放内存 + WAL；只有跨检查点仍未裁决的，才在检查点时落成 `pg_twophase` 文件（`twophase.c:44`）。恢复时两个来源（WAL 中的 PREPARE 记录 + pg_twophase 文件）合并重建。

---

## 5. 风险与运维

prepared 事务**长期持有锁、压低 xmin horizon**（阻止 VACUUM 回收，见 [32-mvcc-snapshot](32-mvcc-snapshot.md)）。若协调者宕机、留下"孤儿" prepared 事务，会持续占锁、导致膨胀，必须人工 `ROLLBACK PREPARED` 清理。因此 `max_prepared_transactions` 默认为 0（关闭），需显式开启。

---

## 6. 设计模式

- **状态外置（内存 → 共享内存/持久文件）**：把本属于 backend 私有的事务状态序列化到共享内存 + WAL + pg_twophase，使事务脱离进程与重启存活——分布式原子提交的物理基础。
- **承诺协议（prepared 即保证可提交）**：第一阶段做完所有可能失败的检查并锁定资源，使第二阶段成为不会失败的纯执行——这是 2PC 协议正确性的语义核心。
- **裁决与执行分离**：谁裁决（任意连接 COMMIT/ROLLBACK PREPARED）与谁准备（原 backend）解耦，靠 gid 寻址。
- **双来源重建（WAL + 状态文件）**：近期事务靠 WAL、跨检查点的靠文件，恢复时合并——以 I/O 优化换持久性，复用恢复机制。
- **虚拟 PGPROC 占位**：用一个无 backend 的 PGPROC 让 prepared 事务在 ProcArray 中继续"存在"（持 XID、持锁），复用既有并发机制而非另造一套。

---

## 7. 架构编排

```
第一阶段（原 backend）：
  PREPARE TRANSACTION 'gid'
    → PrepareTransaction                                     twophase.c
        ├─ MarkAsPreparing（共享内存 GlobalTransaction）      twophase.c:365
        ├─ 序列化锁/失效/待删文件/子事务
        ├─ 写 PREPARE WAL 记录（持久承诺）
        └─ 锁/XID 转交 GlobalTransaction（CLOG 仍 in-progress）
  连接可断开；锁继续被持有

第二阶段（任意 backend）：
  COMMIT PREPARED 'gid' → 找 GlobalTransaction → 写 COMMIT WAL + CLOG COMMITTED
      → 发失效/删文件/放锁 → 删 pg_twophase 文件
  ROLLBACK PREPARED 'gid' → CLOG ABORTED + abort 副作用

崩溃恢复：
  RecoverPreparedTransactions（扫 pg_twophase + WAL PREPARE 记录）
      → 重建 GlobalTransaction + 重新获取锁 → 悬而未决等裁决
```

---

## 8. 动手探索

```sql
-- 需先开启
-- postgresql.conf: max_prepared_transactions = 10  （改后重启）
SHOW max_prepared_transactions;

BEGIN;
  INSERT INTO t VALUES (1);
PREPARE TRANSACTION 'tx1';        -- 连接可断开，锁仍在

-- 查看悬挂的 prepared 事务
SELECT gid, prepared, owner, database FROM pg_prepared_xacts;

COMMIT PREPARED 'tx1';            -- 或 ROLLBACK PREPARED 'tx1';
```

```bash
ls $PGDATA/pg_twophase/          # 跨检查点未裁决的状态文件
```

---

## 相关模块

- 普通事务流程：[30-xact](30-xact.md)
- WAL 持久化：[33-wal](33-wal.md)
- 恢复时重建：[34-recovery-checkpoint](34-recovery-checkpoint.md)
- 锁的持有：[../concurrency/22-heavyweight-lock](../concurrency/22-heavyweight-lock.md)
