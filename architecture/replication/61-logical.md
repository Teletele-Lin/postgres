# 逻辑解码与发布订阅

> 源码：`src/backend/replication/logical/`（`logical.c` `decode.c` `reorderbuffer.c` `snapbuild.c`）、`pgoutput/`
> 对比：[60-physical](60-physical.md)（物理复制）

---

## 1. 职责

把 WAL 中的**物理变更**解码成**逻辑变更**（"表 t 的某行被 INSERT 成 (1,'a')"），按事务提交顺序输出给消费者。逻辑复制传输的是**行级逻辑操作**而非物理块，因而：
- 主备**可异构**：不同大版本、不同物理布局、甚至选择性复制部分表/列/行。
- 支持**双向/多源**复制、把变更喂给外部系统（CDC、消息队列）。

内置的发布订阅（publication/subscription）是逻辑复制的开箱即用形态。

物理 vs 逻辑：物理复制整个集群、字节级镜像、备库只读；逻辑复制按表粒度、行级语义、目标库可写。

---

## 2. 三大组件

逻辑解码的难点在于：WAL 是**物理的、按写入顺序、含未提交事务交织**的，要还原成**逻辑的、按提交顺序、只含已提交**的变更流，还要能在解码时正确读取当时的表结构。三个组件协作解决：

### 2.1 解码（decode.c）

`LogicalDecodingProcessRecord()`（`decode.c:89`）逐条处理 WAL 记录：按 `xl_rmid` 分发（`rm_decode`，见 [../transaction/33-wal](../transaction/33-wal.md) §2.3），把 heap 的 insert/update/delete 记录解析出"哪个表、什么行、什么操作"，交给 ReorderBuffer 暂存（`ReorderBufferProcessXid`，`decode.c:121`）。

### 2.2 ReorderBuffer（reorderbuffer.c）

WAL 里不同事务的记录是**交织**的（多 backend 并发写），且含最终会回滚的事务。`ReorderBuffer` 按 XID 把变更分桶暂存，**直到读到该事务的 COMMIT 记录**，才按提交顺序把它的全部变更"重放"给 output plugin；读到 ABORT 则丢弃。这样消费者只看到**已提交、按提交序**的变更。大事务超内存阈值时溢出到磁盘（或开启流式 `streaming` 在提交前就分批发出）。

### 2.3 SnapBuild（snapbuild.c）

解码时要读**当时的系统表**来知道表结构（列名、类型）——但表结构可能在解码点之后已被 DDL 改过。`SnapBuild` 从 WAL 重建**历史快照**（`SNAPSHOT_HISTORIC_MVCC`，见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)），让解码每个变更时能看到"该变更发生时刻"的 catalog 状态。这要求相关 catalog 的旧版本不被 VACUUM 过早回收——由逻辑复制槽的 `catalog_xmin` 保护。逻辑解码刚启动时需要等待一个一致的快照点才能开始（找到所有在建快照前已结束的事务）。

---

## 3. Output Plugin

`logical.c` 定义 output plugin 回调接口，把"解码出的逻辑变更"转成具体输出格式：

```c
pg_decode_startup / pg_decode_begin / pg_decode_change /
pg_decode_truncate / pg_decode_commit / pg_decode_shutdown ...
```

- **pgoutput**（`pgoutput/pgoutput.c`）：内置插件，按 PostgreSQL 逻辑复制协议输出，供订阅端 apply worker 消费。发布订阅用的就是它。
- 自定义插件（如 `wal2json`、`test_decoding`）可输出 JSON 等格式，用于 CDC、调试。

---

## 4. 内置发布订阅

```sql
-- 发布端（primary）
CREATE PUBLICATION pub FOR TABLE orders, customers;

-- 订阅端（另一个库）
CREATE SUBSCRIPTION sub
  CONNECTION 'host=... dbname=...' PUBLICATION pub;
```

### 4.1 系统表与进程

- `pg_publication` / `pg_publication_rel`：发布了哪些表（可带行过滤、列列表）。
- `pg_subscription` / `pg_subscription_rel`：订阅配置与每表同步状态。
- **logical replication launcher**（postmaster 子进程，见 [../process/51-background-procs](../process/51-background-procs.md)）：为每个订阅启动 **apply worker**。
- **apply worker**（订阅端）：连发布端的 walsender（逻辑模式），接收 pgoutput 流，在本地执行对应的 INSERT/UPDATE/DELETE。
- **tablesync worker**：订阅初建时为每个表做初始数据拷贝（COPY），追上后切换到流式 apply。

### 4.2 流程

```
发布端：walsender（逻辑模式）
  → 逻辑解码（decode + ReorderBuffer + SnapBuild）
  → pgoutput 编码 → 复制协议发送
订阅端：apply worker
  → 接收 → 解码协议 → 在本地表执行行变更 → 推进 confirmed_flush（逻辑槽）
```

逻辑复制槽的 `confirmed_flush`/`restart_lsn` 记录订阅消费进度，保护未确认的 WAL 与 `catalog_xmin`（见 [60-physical](60-physical.md) §3）。

---

## 5. 限制与注意

- 默认只复制 DML，**不复制 DDL**（表结构变更需两端各自执行或用扩展）。
- 需要表有**副本标识**（replica identity，通常是主键）来定位 UPDATE/DELETE 的目标行；无主键表需设 `REPLICA IDENTITY FULL`。
- 冲突（如订阅端已有同主键行）会让 apply worker 报错暂停，需人工干预。
- 逻辑槽长时间不消费 → `catalog_xmin` 卡住 → catalog 膨胀。

---

## 6. 设计模式

- **物理日志 → 逻辑变更的重构**：从面向恢复的 WAL 反向重建行级语义，复用同一份日志服务两种复制——避免为逻辑复制单设写入通道。
- **重排到提交序（ReorderBuffer）**：把交织的、含未提交的物理流，缓冲重排为"已提交、按提交序"的干净逻辑流，屏蔽并发与回滚的复杂性。
- **历史快照解码（SnapBuild）**：用 WAL 重建历史 catalog 视图，使"按变更发生时的表结构解读变更"成为可能，并以 `catalog_xmin` 保护所需旧版本。
- **Output Plugin 策略化**：解码内核与输出格式解耦，一套解码喂给 pgoutput/JSON/自定义，支撑复制与 CDC 多场景。
- **launcher/worker 编排**：用 launcher + per-subscription apply worker + per-table sync worker 分层，复用 bgworker 与槽机制。

---

## 7. 架构编排

```
发布端：
  walsender（逻辑）→ LogicalDecodingContext
    └─ LogicalDecodingProcessRecord（按 rmid 解码）                decode.c:89
         ├─ ReorderBuffer：按 XID 暂存 → 读到 COMMIT 才按序输出     reorderbuffer.c
         ├─ SnapBuild：历史快照读 catalog                          snapbuild.c
         └─ output plugin（pgoutput）编码 → 复制协议
    逻辑槽：restart_lsn + catalog_xmin 保护 WAL 与 catalog 旧版本
订阅端：
  logical replication launcher → apply worker（每订阅）            process/51
    ├─ tablesync worker：初始 COPY
    └─ 流式接收 pgoutput → 本地执行行变更 → 推进 confirmed_flush
```

---

## 8. 动手探索

```sql
-- 不建订阅，直接看解码（test_decoding 插件）
SELECT * FROM pg_create_logical_replication_slot('s1', 'test_decoding');
INSERT INTO t VALUES (1,'a');
SELECT data FROM pg_logical_slot_get_changes('s1', NULL, NULL);
--  BEGIN / table public.t: INSERT: id[integer]:1 c[text]:'a' / COMMIT

-- 发布订阅状态
SELECT * FROM pg_publication;
SELECT subname, subenabled FROM pg_subscription;
SELECT slot_name, catalog_xmin, confirmed_flush_lsn FROM pg_replication_slots
WHERE slot_type='logical';

DROP_REPLICATION_SLOT -- 用完清理：SELECT pg_drop_replication_slot('s1');
```

---

## 相关模块

- WAL 与 RMGR 解码：[../transaction/33-wal](../transaction/33-wal.md)
- 历史快照：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
- 物理复制对比：[60-physical](60-physical.md)
- apply/launcher 进程：[../process/51-background-procs](../process/51-background-procs.md)
