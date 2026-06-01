# 检查点与崩溃恢复

> 源码：`src/backend/access/transam/xlog.c`、`xlogrecovery.c`、`src/backend/postmaster/checkpointer.c`
> 上游：[33-wal](33-wal.md)（WAL 写入与格式）

---

## 1. 职责

- **检查点（checkpoint）**：周期性地把所有脏页刷盘，并在 WAL 中记一个"此点之前的修改都已落盘"的标记，从而**界定崩溃恢复的起点**、允许回收旧 WAL。
- **崩溃恢复（crash recovery）**：重启后从上次检查点开始重放 WAL，把内存中未落盘的修改重建出来，使数据库回到崩溃前的一致状态。
- 二者是同一枚硬币的两面：检查点决定恢复要从哪开始、要重放多少。

---

## 2. 检查点

### 2.1 redo point —— 恢复的起点

检查点最关键的产物是 **redo point**：检查点**开始时**的 WAL 插入位置（`checkPoint.redo`，`xlog.c:6119`）。恢复必须从 redo point 开始重放——因为 redo point 之后的修改可能还没落盘。注意是检查点**开始**而非结束时的 LSN，因为检查点刷盘期间仍有新修改在发生。

### 2.2 检查点流程

`CreateCheckPoint()`（`xlog.c`）→ `CheckPointGuts()`（`xlog.c:703`）：

```
1. 记录 redo point = 当前 WAL 插入位置
2. CheckPointGuts：刷各子系统
   ├─ CheckPointBuffers：把所有脏数据页写盘（BufferSync，排序后批量，遵守 WAL 先行）
   ├─ CheckPointCLOG / SUBTRANS / MultiXact：刷 SLRU
   ├─ CheckPointReplicationSlots / Origins
   └─ ProcessSyncRequests：执行延迟的 fsync（md.c 攒的脏段，见 storage/17）
3. 写 checkpoint WAL 记录（含 redo point、nextXID、oldestXID 等）
4. 更新 pg_control：checkPoint = redo point（xlog.c:5599），记最新检查点位置
5. 回收/归档 redo point 之前不再需要的 WAL 段
```

`pg_control`（`global/pg_control`）是恢复的"引导扇区"：记录最近检查点位置、时间线、系统状态（崩溃前是否干净关闭）等。

### 2.3 触发时机与平滑

- **时间触发**：`checkpoint_timeout`（默认 5min）。
- **WAL 量触发**：写满 `max_wal_size` 的一定比例。
- **手动**：`CHECKPOINT` 命令；关库时。

由独立的 **checkpointer 进程**执行（见 [../process/51-background-procs](../process/51-background-procs.md)）。为避免检查点刷盘造成 I/O 尖峰，刷脏页按 `checkpoint_completion_target`（默认 0.9）**摊匀**到整个检查点间隔——把一次性大批写打散成持续的缓流。

---

## 3. 崩溃恢复

### 3.1 启动判定

重启时 **startup 进程**读 `pg_control`：若状态不是"干净关闭"（`DB_SHUTDOWNED`），说明上次崩溃 → 进入恢复。否则跳过。

### 3.2 REDO 主循环

`PerformWalRecovery()`（`xlogrecovery.c`）从 redo point 开始顺序重放：

```
LSN = pg_control.checkPoint.redo
while (record = ReadRecord(LSN)) {          // xlogreader 解析下一条，校验 CRC
    if (record 改了某些 block)
        for each block in record:
            读入该页（不在则从磁盘读）
            if (page->pd_lsn >= record_LSN)
                跳过；                        // 该页已包含此修改（已落盘）—— 幂等关键
            else
                RmgrTable[record->xl_rmid].rm_redo(record);  // 重放（按 RMGR 分发）
                PageSetLSN(page, record_LSN);
    推进到 consistent point 后，hot standby 可开始接受只读查询
}
```

要点：
- **页 LSN 比较实现幂等重放**（`xlogrecovery.c`）：若页的 `pd_lsn` 已 ≥ 记录 LSN，说明崩溃前该修改已落盘，跳过。于是重放任意次都安全——恢复可被再次中断再恢复。
- **FPI 覆盖撕裂页**：记录若带 full-page image，直接整页覆盖，无视原页内容（见 [33-wal](33-wal.md) §3.3）。
- **`rm_redo` 分发**：每条记录交给其资源管理器重放，heap 记录由 `heap_redo` 重建元组，btree 记录由 `btree_redo` 重建索引页，等等。

### 3.3 恢复的三阶段一致性

恢复推进到 **consistent point**（所有备份/检查点期间的不一致都已被 WAL 抹平）后：
- 崩溃恢复：继续重放到 WAL 末尾，然后开放读写。
- **Hot Standby**（备库）：到达一致点后即可接受**只读查询**，同时继续重放主库源源不断的 WAL（见 [../replication/60-physical](../replication/60-physical.md)）。

### 3.4 恢复后的收尾

重放到 WAL 末尾后：回滚崩溃时未提交的事务（其 XID 在 CLOG 中无 COMMITTED 标记，默认视为 abort，留下的脏元组待 VACUUM 清理）、`RecoverPreparedTransactions()` 重建预备事务（见 [35-twophase](35-twophase.md)）、起新时间线、做一次 end-of-recovery 检查点。

---

## 4. 时间线（timeline）与 PITR

- **时间线 ID**：每次从备份恢复并允许写、或备库提升为主库，都开一条新时间线，避免 WAL 历史分叉混淆。WAL 段文件名前 8 位就是时间线。
- **PITR（Point-In-Time Recovery）**：基础备份 + 归档 WAL，可恢复到任意时刻（`recovery_target_time/lsn/xid`）。恢复重放到目标点即停，开新时间线。这与崩溃恢复共用同一套 REDO 机制，只是数据来自归档而非本地 `pg_wal`。

---

## 5. 设计模式

- **检查点界定恢复窗口**：用 redo point 把"需要重放的 WAL"限定在一个有界区间，否则要从创库重放至今。检查点频率是"恢复时长"与"运行时刷盘开销"的权衡。
- **幂等重放（页 LSN 比较）**：让 REDO 可重复、可中断重启，是崩溃恢复鲁棒性的核心——恢复本身崩溃也能再恢复。
- **RMGR 多态重放**：恢复不懂任何具体存储格式，全靠 `rm_redo` 分发，与 WAL 写入侧对称。
- **pg_control 作引导真相源**：把"从哪恢复、是否干净关闭、什么时间线"集中在一个小而关键的控制文件。
- **刷盘摊匀（completion target）**：把检查点的突发 I/O 摊平到整个周期，避免周期性卡顿。
- **时间线防分叉**：用单调时间线 ID 给可能分叉的 WAL 历史编号，使备份/提升/恢复的因果清晰。

---

## 6. 架构编排

```
正常运行：
  checkpointer ──周期/WAL量触发──> CreateCheckPoint
      ├─ redo point = 当前插入 LSN
      ├─ CheckPointGuts：刷脏页(BufferSync) + SLRU + fsync 延迟段       xlog.c:703
      ├─ 写 checkpoint WAL 记录
      └─ 更新 pg_control，回收旧 WAL                                    xlog.c:5599

崩溃重启：
  startup 进程读 pg_control（非干净关闭 → 恢复）
      └─ PerformWalRecovery：从 checkPoint.redo 顺序重放               xlogrecovery.c
            ReadRecord → 页 LSN 比较（幂等）→ rm_redo 分发 / FPI 覆盖
            → consistent point（Hot Standby 可读）
            → 重放到末尾 → 回滚未提交 + 恢复预备事务 → end-of-recovery 检查点
```

---

## 7. 动手探索

```sql
SELECT * FROM pg_control_checkpoint();        -- redo LSN、时间线、nextXID 等
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint
FROM pg_stat_checkpointer;                    -- 检查点频率与刷盘量
CHECKPOINT;                                    -- 手动触发
SHOW checkpoint_timeout; SHOW max_wal_size; SHOW checkpoint_completion_target;
```

```bash
pg_controldata $PGDATA        # 看 pg_control：最近检查点、redo 位置、状态、时间线
# 崩溃恢复日志：server log 中 "database system was not properly shut down;
#               automatic recovery in progress" → "redo starts at" → "redo done"
```

---

## 相关模块

- WAL 写入与 RMGR：[33-wal](33-wal.md)
- 脏页刷盘：[../storage/16-buffer-manager](../storage/16-buffer-manager.md)
- checkpointer 进程：[../process/51-background-procs](../process/51-background-procs.md)
- 备库回放：[../replication/60-physical](../replication/60-physical.md)
- 预备事务恢复：[35-twophase](35-twophase.md)
