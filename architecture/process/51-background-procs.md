# 后台辅助进程

> 源码：`src/backend/postmaster/`（`checkpointer.c` `bgwriter.c` `walwriter.c` `autovacuum.c` `pgarch.c` `startup.c` 等）
> 父进程：[50-postmaster](50-postmaster.md)

---

## 1. 职责

Postmaster fork 出一批**专职后台进程**，把"周期性维护"从 backend 主路径剥离，使前台查询更平稳。它们各管一摊，通过共享内存协作、靠 latch 被唤醒（见 [../concurrency/20-lock-overview](../concurrency/20-lock-overview.md) §3）。

---

## 2. 进程清单与职责

| 进程 | 源码 | 职责 | 唤醒方式 |
|------|------|------|---------|
| **startup** | `startup.c` | 崩溃恢复 / 备库回放 WAL（仅恢复期，恢复完即退出或转为持续回放） | 持续 |
| **checkpointer** | `checkpointer.c` | 写检查点、刷脏页、执行延迟 fsync | 定时/WAL 量/请求 |
| **background writer** | `bgwriter.c` | 平时预刷脏页，平滑 I/O，减轻检查点与 backend 换页压力 | `bgwriter_delay` |
| **walwriter** | `walwriter.c` | 周期 flush WAL buffer，让异步提交也能较快落盘 | `wal_writer_delay` |
| **autovacuum launcher** | `autovacuum.c` | 监控各库膨胀/回卷，按需 fork autovacuum worker | 定时 |
| **autovacuum worker** | `autovacuum.c` | 实际执行 VACUUM/ANALYZE | 被 launcher 启动 |
| **archiver** | `pgarch.c` | 对完成的 WAL 段执行 `archive_command`/`archive_library` | WAL 段切换 |
| **walsender** | `replication/walsender.c` | 向备库/逻辑订阅发送 WAL（每连接一个） | 见 [../replication/60](../replication/60-physical.md) |
| **walreceiver** | `replication/walreceiver.c` | 备库接收主库 WAL（仅 standby） | 见 [../replication/60](../replication/60-physical.md) |
| **logical replication launcher / apply worker** | `replication/logical/` | 启动并运行订阅端 apply | 见 [../replication/61](../replication/61-logical.md) |
| **io workers** | `storage/aio/` | AIO `worker` 模式下代为执行阻塞 I/O | 见 [../storage/17](../storage/17-smgr-forks.md) |
| **stats collector → 现为共享内存** | `utils/activity/` | 统计（PG15 起改为共享内存，无独立进程） | — |

---

## 3. 几个关键进程详解

### 3.1 checkpointer

把检查点从 backend 中独立出来（历史上检查点曾由 bgwriter 兼职）。详见 [../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md)。要点：刷脏页按 `checkpoint_completion_target` 摊匀避免 I/O 尖峰；还集中执行 md.c 攒下的延迟 fsync 请求（`ProcessSyncRequests`，见 [../storage/17](../storage/17-smgr-forks.md)）。backend 提交时只需保证 WAL 落盘，数据页落盘交给 checkpointer/bgwriter，这是"WAL 先行"得以高效的组织前提。

### 3.2 background writer

与 checkpointer 分工：checkpointer 是"周期性把所有脏页刷一遍"，bgwriter 是"平时持续地把时钟扫描即将淘汰的脏页提前刷掉"，让 backend 需要换页时更可能拿到干净 victim（见 [../storage/16-buffer-manager](../storage/16-buffer-manager.md)），减少 backend 自己刷脏页的停顿。

### 3.3 autovacuum

- **launcher**：常驻，按 `autovacuum_naptime` 轮询，根据 `pg_stat` 的死元组数、XID 年龄（防回卷，见 [../transaction/31-clog-slru](../transaction/31-clog-slru.md)）决定哪些库/表需要处理，向 postmaster 请求 fork worker。
- **worker**：连到目标库，对超阈值的表跑 VACUUM（回收死元组、冻结老 XID、更新 VM/FSM）和 ANALYZE（更新统计供优化器）。受 `autovacuum_max_workers` 并发上限与 cost-based delay（限速避免冲击前台）约束。

autovacuum 是 PostgreSQL "免运维"的关键，但配置不当（阈值太高、worker 太少、长事务压低 horizon）会导致膨胀与回卷风险，是运维重点。

---

## 4. 协作机制

- **共享内存通信**：各进程通过共享内存的请求结构传递任务（如 backend 请求检查点 → 设共享标志 + SetLatch(checkpointer)）。
- **Latch 唤醒**：每个后台进程主循环是 `WaitLatch(超时)` + 处理 + 循环。别的进程 `SetLatch` 即可立刻唤醒它干活，否则按超时周期性醒来。
- **postmaster 监管**：任一辅助进程异常退出，postmaster 按角色重要性决定重启它还是触发全体重置（见 [50-postmaster](50-postmaster.md)）。

---

## 5. 设计模式

- **关注点分离（专职进程）**：把检查点、脏页预刷、WAL flush、VACUUM、归档各设专职进程，从前台查询路径剥离周期性/批量工作，使延迟更可预测。
- **生产者-消费者 + Latch**：backend 产生请求（检查点/fsync/唤醒），后台进程消费；用 latch 做事件驱动的睡眠/唤醒，避免轮询空转。
- **launcher/worker 两级**：autovacuum 用常驻 launcher 决策 + 临时 worker 执行，按需伸缩并发，复用 postmaster 的 fork 监管。
- **限速维护（cost-based delay）**：后台维护主动限速，避免与前台争 I/O——后台让位前台的设计取向。
- **postmaster 统一监管**：所有辅助进程的生死纳入 postmaster 状态机，崩溃可统一重置。

---

## 6. 架构编排

```
postmaster（PM_RUN）
  ├─ startup（恢复期）→ 恢复完成
  ├─ checkpointer  ←SetLatch← backend 请求检查点 / 定时 / WAL 量
  │     └─ 刷脏页(BufferSync) + 延迟 fsync + 写检查点（见 transaction/34）
  ├─ bgwriter      → 持续预刷即将淘汰的脏页（见 storage/16）
  ├─ walwriter     → 周期 flush WAL buffer（见 transaction/33）
  ├─ autovacuum launcher → fork worker → VACUUM/ANALYZE（见 storage/11）
  ├─ archiver      → archive_command（WAL 段切换时）
  └─ walsender/逻辑 launcher → 复制（见 replication/60,61）
```

---

## 7. 动手探索

```sql
SELECT pid, backend_type, wait_event FROM pg_stat_activity
WHERE backend_type <> 'client backend';     -- 看各后台进程在等什么

SELECT * FROM pg_stat_checkpointer;          -- 检查点统计
SELECT * FROM pg_stat_bgwriter;              -- bgwriter 刷盘统计
SELECT schemaname, relname, last_autovacuum, autovacuum_count, n_dead_tup
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

SHOW autovacuum_naptime; SHOW autovacuum_max_workers; SHOW bgwriter_delay;
```

---

## 相关模块

- 父进程：[50-postmaster](50-postmaster.md)
- 检查点：[../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md)
- 脏页刷盘：[../storage/16-buffer-manager](../storage/16-buffer-manager.md)
- VACUUM：[../storage/11-heap](../storage/11-heap.md)
- 复制：[../replication/60-physical](../replication/60-physical.md)
