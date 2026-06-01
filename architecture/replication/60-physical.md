# 物理流复制

> 源码：`src/backend/replication/{walsender,walreceiver,slot,syncrep}.c`、`libpqwalreceiver/`
> 数据源：[../transaction/33-wal](../transaction/33-wal.md) 的 WAL

---

## 1. 职责

把主库（primary）的 **WAL 字节流**实时发送到备库（standby），备库回放 WAL 重建出与主库**字节级一致**的副本。用途：高可用（故障切换）、读扩展（Hot Standby 只读查询）、PITR 基础。

"物理"指传输的是 WAL 物理记录（块级变更），备库是主库的精确物理镜像（同样的 relfilenode、同样的块布局）。与之相对的是逻辑复制（见 [61-logical](61-logical.md)）。

---

## 2. 进程拓扑

```
   primary                                   standby
 ┌──────────┐   WAL 流（libpq 复制协议）   ┌───────────┐
 │ walsender│ ───────────────────────────> │walreceiver│
 └──────────┘                              └─────┬─────┘
   ↑ 读 WAL                                       │ 写本地 pg_wal
 pg_wal/                                          ▼
   ↑ backend 写 WAL                          ┌───────────┐
 backend                                     │  startup  │ 回放 WAL（持续 redo）
                                             └───────────┘
                                                  │ Hot Standby
                                             只读 backend 查询
```

- **walsender**（`walsender.c`）：主库上每个复制连接一个。备库发来 `START_REPLICATION`（`walsender.c:14`）后进入流式发送循环 `WalSndLoop`（`walsender.c:286`），`XLogSendPhysical()`（`walsender.c:290`）把新 WAL 发出去。
- **walreceiver**（`walreceiver.c`）：备库上一个，`WalReceiverMain` 连主库、接收 WAL、写本地 `pg_wal`，通知 startup 进程回放。
- **startup**：备库上持续回放 WAL（崩溃恢复的"持续版"，见 [../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md)）。

---

## 3. 复制槽（replication slot）

`slot.c`。没有槽时，主库不知道备库消费到哪，可能在备库还没收到前就回收/归档了需要的 WAL（备库追不上 → 断链）。**复制槽**解决之：

- 槽持久记录该消费者的进度：`restart_lsn`（仍需保留的最老 WAL 位置）、`confirmed_flush`（逻辑槽用）。
- 主库**保证不回收** `restart_lsn` 之后的 WAL，直到消费者确认。
- 物理槽还可记录备库的 `xmin`（`hot_standby_feedback`），防止主库 VACUUM 回收备库查询仍需的旧元组版本。

代价：备库长时间掉线会让主库 WAL 无限堆积（`max_slot_wal_keep_size` 可设上限）。槽是物理与逻辑复制共用的进度追踪机制。

---

## 4. 同步 vs 异步复制

`syncrep.c`：

- **异步**（默认）：主库 backend 提交时不等备库，写完本地 WAL 即返回。备库稍有延迟；主库突然故障可能丢失最近未传输的事务。
- **同步**：`synchronous_commit` + `synchronous_standby_names` 配置下，主库提交时**等待指定备库确认收到/落盘/回放** WAL 后才告诉客户端"已提交"。保证零数据丢失（在确认级别内），代价是提交延迟受网络与备库影响。

`synchronous_commit` 的级别（`off`/`local`/`remote_write`/`on`/`remote_apply`）精细控制"等到什么程度"——从不等备库到等备库回放完。同步提交在 [../transaction/30-xact](../transaction/30-xact.md) 的 `CommitTransaction` 中等待。

---

## 5. Hot Standby

备库回放 WAL 的同时**接受只读查询**（`hot_standby=on`）。挑战与机制：
- **回放冲突**：主库的 VACUUM/DDL 回放可能要删除/锁定备库正在被只读查询使用的数据。冲突时按 `max_standby_streaming_delay` 等待，超时则**取消备库查询**（`PROCSIG_RECOVERY_CONFLICT_*`，见 [../process/52-shmem-ipc](../process/52-shmem-ipc.md)），让回放优先（保证备库不落后）。
- **快照**：备库的快照标 `takenDuringRecovery`，可见性判断要考虑主库传来的运行中事务信息（`KnownAssignedXids`，由 WAL 中的 `Standby` 记录维护）。
- **`hot_standby_feedback`**：备库把自己的 xmin 反馈给主库（经物理槽），让主库 VACUUM 不回收备库查询需要的版本——以主库轻微膨胀换备库查询不被取消。

---

## 6. 级联与故障切换

- **级联复制**：备库自己也可跑 walsender，把 WAL 转发给下游备库（standby of standby）。
- **提升（promote）**：`pg_promote()` / 触发文件让备库结束恢复、成为可写主库，开**新时间线**（见 [../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md) §4），原下游可经 `pg_rewind` 重新跟随。

---

## 7. 设计模式

- **日志即复制（WAL 复用）**：复制不另造数据通道，直接流式传输已有的 WAL——崩溃恢复与复制共用同一套 redo 机制，备库回放 == 永不结束的崩溃恢复。
- **进度槽（slot）解耦生产消费**：用持久化的 `restart_lsn` 让主库精确知道"还需保留哪些 WAL / 哪些旧元组"，把消费者进度反作用于主库的资源回收。
- **同步级别可调**：用 `synchronous_commit` 在"零丢失"与"低延迟"间连续取舍，而不改一致性。
- **回放优先 + 冲突取消（Hot Standby）**：备库读不能拖垮回放，冲突时牺牲查询保证副本不落后——副本一致性优先于备库可用性。
- **反馈回路（hot_standby_feedback）**：备库把 horizon 需求反推给主库，用主库膨胀换备库查询稳定，是分布式 horizon 协调。
- **时间线防分叉**：提升产生新时间线，使主备角色变更后的 WAL 因果不混淆。

---

## 8. 架构编排

```
primary：
  backend 写 WAL（见 transaction/33）→ pg_wal
  walsender ← START_REPLICATION ← standby
     └─ XLogSendPhysical：读 pg_wal 新 WAL → libpq 复制协议发送        walsender.c:290
     └─ 复制槽 restart_lsn 限制 WAL 回收 / hot_standby_feedback 限制 VACUUM
  syncrep：同步模式下 CommitTransaction 等备库确认                      syncrep.c

standby：
  walreceiver：接收 WAL → 写本地 pg_wal → 通知 startup                 walreceiver.c
  startup：持续回放 WAL（redo）                                         见 transaction/34
  Hot Standby：只读 backend 查询；回放冲突 → 等待/取消查询
```

---

## 9. 动手探索

```sql
-- 主库：复制连接与延迟
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag, sync_state
FROM pg_stat_replication;

-- 复制槽
SELECT slot_name, slot_type, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;

-- 备库：恢复状态与延迟
SELECT pg_is_in_recovery(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
SELECT now() - pg_last_xact_replay_timestamp() AS replay_delay;

SHOW synchronous_standby_names; SHOW hot_standby_feedback;
```

---

## 相关模块

- WAL：[../transaction/33-wal](../transaction/33-wal.md)
- 备库回放 = 恢复：[../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md)
- 逻辑复制：[61-logical](61-logical.md)
- 同步提交：[../transaction/30-xact](../transaction/30-xact.md)
- 回放冲突信令：[../process/52-shmem-ipc](../process/52-shmem-ipc.md)
