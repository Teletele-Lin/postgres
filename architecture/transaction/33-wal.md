# WAL — 预写日志：写入与记录格式

> 源码：`src/backend/access/transam/{xlog,xloginsert,xlogreader}.c`、`src/include/access/{xlogrecord,rmgr}.h`
> 配套：[34-recovery-checkpoint](34-recovery-checkpoint.md)（恢复与检查点）

---

## 1. 职责

WAL（Write-Ahead Log，又称 xlog）是 PostgreSQL **持久性与崩溃可恢复性**的基石，也是物理复制与 PITR 的数据源。核心铁律 —— **WAL 先行（Write-Ahead Logging）**：

> 任何对数据页的修改，其描述该修改的 WAL 记录必须**先于**该脏页落盘持久化。

于是即使数据页还在内存里就崩溃了，重启时可从 WAL 重放出这些修改。WAL 是顺序追加写（快），数据页是随机写（慢），WAL 先行把"随机写的持久化"延迟、批量化，是数据库性能与可靠性兼得的关键。

---

## 2. 核心数据结构

### 2.1 XLogRecord —— 记录头

`src/include/access/xlogrecord.h:41`：

```c
typedef struct XLogRecord {
    uint32        xl_tot_len;  // 整条记录总长度                     xlogrecord.h:43
    TransactionId xl_xid;      // 所属事务 XID                        xlogrecord.h:44
    XLogRecPtr    xl_prev;     // 指向本事务上一条 WAL 记录的 LSN      xlogrecord.h:45
    uint8         xl_info;     // 低4位 XLOG 内部标志；高4位 RMGR 自定义 xlogrecord.h:46
    RmgrId        xl_rmid;     // 资源管理器 ID（RM_HEAP_ID 等）        xlogrecord.h:47
    /* 2 字节对齐填充 */
    pg_crc32c     xl_crc;      // 整条记录的 CRC32C 校验和              xlogrecord.h:49
    /* 之后是 XLogRecordBlockHeader(s) + 主数据 */
} XLogRecord;
```

记录头之后是若干 **block 引用**（`XLogRecordBlockHeader`，`xlogrecord.h:103`）+ 主数据。每个 block 引用描述"这条记录改了哪个 `(rel,fork,block)`"，可携带 **full-page image（FPI）** 或仅增量。

### 2.2 LSN（Log Sequence Number）

`XLogRecPtr`（64 位）是 WAL 中的字节偏移，全局单调递增，唯一标识一条记录的位置。每个数据页头 `pd_lsn`（见 [../storage/15-page-layout](../storage/15-page-layout.md)）记"最后修改本页的 WAL 记录 LSN"——这正是 WAL 先行规则的判据与恢复时"是否需要重放"的依据。

### 2.3 资源管理器（RMGR）

`src/include/access/rmgr.h`。WAL 记录按 `xl_rmid` 归属于一个**资源管理器**：`RM_XLOG_ID`、`RM_XACT_ID`、`RM_HEAP_ID`、`RM_HEAP2_ID`、`RM_BTREE_ID`、`RM_GIN_ID`、`RM_GIST_ID`、`RM_STANDBY_ID` 等。每个 RMGR 注册到 `RmgrTable[]`，实现回调：

```c
typedef struct RmgrData {
    const char *rm_name;
    void (*rm_redo)(XLogReaderState *record);   // 重放（恢复时调用）
    void (*rm_desc)(StringInfo, XLogReaderState *); // 人类可读描述（pg_waldump）
    void (*rm_decode)(...);                      // 逻辑解码
    ...
} RmgrData;
```

恢复/解码时按 `RmgrTable[record->xl_rmid].rm_redo/rm_decode` 分发——**资源管理器分发模式**贯穿 WAL/恢复/逻辑复制（见 [00-overview](../00-overview.md) §7）。

---

## 3. 核心算法：WAL 写入

### 3.1 三步注册 + 插入

访问方法（如 `heap_insert`）在修改页面的临界区内构造 WAL 记录：

```c
XLogBeginInsert();                                       // xloginsert.c:153  开始组装
XLogRegisterData(&xlrec, SizeOfHeapInsert);              // xloginsert.c:372  注册主数据
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);          // xloginsert.c:246  注册被改页面
XLogRegisterData(tupledata, len);                        //                   注册元组数据
recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_INSERT);       // xloginsert.c:482  组装并插入
PageSetLSN(page, recptr);                                //                   写回页 LSN
```

`XLogRegisterBuffer` 决定是否带 **full-page image**：见 §3.3。

### 3.2 XLogInsert → XLogInsertRecord

`XLogInsert()`（`xloginsert.c:482`）把注册的数据组装成 `XLogRecData` 链、算 CRC，调 `XLogInsertRecord()`（`xlog.c:784`）：

1. 取一个 **WALInsertLock**（多把，`NUM_XLOGINSERT_LOCKS`，见 [../concurrency/21-lwlock](../concurrency/21-lwlock.md)）—— 多把锁允许多个 backend **并行**预留 WAL 空间。
2. 原子推进 `Insert->CurrBytePos` 预留本记录的字节区间（得到本记录 LSN）。
3. 把记录字节拷进 **WAL Buffer**（共享内存环形缓冲）对应位置。
4. 释放 WALInsertLock，返回记录末尾 LSN。

此时记录在内存 WAL Buffer，**尚未落盘**。

### 3.3 Full-Page Image 与 torn page 防护

操作系统写 8KB 页可能"撕裂"（torn page）——崩溃时半新半旧。防护：**某页在一次检查点后第一次被修改时**，整页镜像（FPI）随 WAL 记录写出。恢复时直接用 FPI 覆盖该页，避开撕裂页。由 `full_page_writes` 控制（默认开）。FPI 是 WAL 体积的大头，故"检查点后首改"才发，且 FPI 会被压缩（`wal_compression`）。

### 3.4 落盘：XLogFlush

WAL Buffer 的内容由以下时机刷到磁盘并 fsync：
- **事务提交**：`synchronous_commit=on` 时 `CommitTransaction` 调 `XLogFlush(commit LSN)` 等待 WAL 落盘后才告诉客户端"已提交"——这是持久性保证点。
- **刷脏页前**（WAL 先行）：buffer manager `FlushBuffer` 前 `XLogFlush(page LSN)`（见 [../storage/16-buffer-manager](../storage/16-buffer-manager.md)）。
- **walwriter** 周期性刷（`wal_writer_delay`）。

`synchronous_commit=off` 允许提交不等 WAL flush（异步提交）：崩溃可能丢最近几个事务，但绝不破坏一致性（数据页仍受 WAL 先行保护），用持久性换吞吐。

### 3.5 WAL 段文件与读取

WAL 在磁盘上是 `pg_wal/` 下的固定大小段文件（默认 16MB，名字编码时间线 + LSN）。`xlogreader.c` 的 `XLogReaderState` 解析 WAL 字节流为一条条记录，校验 CRC，供恢复（[34](34-recovery-checkpoint.md)）、逻辑解码（[../replication/61-logical](../replication/61-logical.md)）、`pg_waldump` 使用。

---

## 4. 设计模式

- **WAL 先行（顺序日志换随机持久化）**：把昂贵的随机数据页持久化延迟、批量，用廉价的顺序日志先固化变更——数据库可靠性的根本范式。
- **资源管理器分发（RMGR）**：WAL 记录自描述归属，回放/描述/解码按 `xl_rmid` 多态分发，新存储模块只需注册 RMGR 即获得 WAL 能力。
- **LSN 作为全局时序与判据**：64 位单调 LSN 既是位置又是版本号，页 `pd_lsn` 把"该页是否已被某记录改过/需否重放"压成一次比较。
- **并行预留 + 环形缓冲**：多把 WALInsertLock + 原子推进字节游标，让多 backend 并行写 WAL，化解单点序列化瓶颈。
- **FPI 防撕裂**：用"检查点后首改发整页"以幂等覆盖应对 OS 非原子写，恢复正确性优先于日志体积。
- **持久性可调（synchronous_commit）**：把"提交是否等 fsync"做成开关，让用户在持久性与吞吐间权衡而不损一致性。

---

## 5. 架构编排

```
访问方法修改页（临界区内）
  XLogBeginInsert → XLogRegisterData/Buffer → XLogInsert(RM_xxx, info)   xloginsert.c
    └─ XLogInsertRecord                                                   xlog.c:784
         ├─ WALInsertLock（多把，并行预留）
         ├─ 原子推进 CurrBytePos → 得 LSN
         └─ 拷入 WAL Buffer（环形共享内存）
  PageSetLSN(page, LSN)；MarkBufferDirty
落盘：
  提交（sync_commit）→ XLogFlush(commitLSN) → fsync pg_wal 段
  刷脏页前 → XLogFlush(pageLSN)（WAL 先行）
  walwriter 周期 flush
下游：xlogreader 解析 → 恢复(rm_redo) / 逻辑解码(rm_decode) / pg_waldump(rm_desc)
```

---

## 6. 动手探索

```sql
SELECT pg_current_wal_lsn(), pg_current_wal_insert_lsn();
SELECT pg_walfile_name(pg_current_wal_lsn());      -- 当前 WAL 段文件名

-- 写放大 / FPI 观察
EXPLAIN (ANALYZE, WAL) INSERT INTO t SELECT generate_series(1,1000);
-- 输出 WAL: records=.. fpi=.. bytes=..

SHOW synchronous_commit;  SHOW full_page_writes;  SHOW wal_compression;
```

```bash
pg_waldump $PGDATA/pg_wal/000000010000000000000001 | head   # rm_desc 输出每条记录
pg_waldump --stats $PGDATA/pg_wal/0000000100000000000000XX   # 按 RMGR 统计
```

---

## 相关模块

- 恢复与检查点：[34-recovery-checkpoint](34-recovery-checkpoint.md)
- WAL 先行的执行者：[../storage/16-buffer-manager](../storage/16-buffer-manager.md)
- 页 LSN：[../storage/15-page-layout](../storage/15-page-layout.md)
- 提交时机：[30-xact](30-xact.md)
- 物理复制消费 WAL：[../replication/60-physical](../replication/60-physical.md)
