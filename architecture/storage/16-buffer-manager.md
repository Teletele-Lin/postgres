# Buffer Manager — 缓冲区管理与时钟扫描

> 源码：`src/backend/storage/buffer/`（`bufmgr.c` `freelist.c` `buf_table.c` `localbuf.c`）
> 头文件：`src/include/storage/bufmgr.h`、`buf_internals.h`

---

## 1. 职责

在磁盘（SMGR）与访问方法之间维护一个**共享内存缓冲池**，缓存最近访问的页面，减少磁盘 I/O。核心职责：
- **页面缓存与查找**：给定 `(rel, fork, block)` 快速找到内存中的页（或读入）。
- **pin/unpin**：防止正在使用的页被换出。
- **脏页管理与回写**：跟踪被修改的页，遵守 **WAL 先行**规则刷盘。
- **替换策略**：缓冲池满时选择淘汰哪个页（时钟扫描）。
- **访问策略环（ring buffer）**：避免大顺序扫描/VACUUM 污染整个缓冲池。

缓冲池大小由 `shared_buffers` 决定，是所有 backend 共享的（在共享内存中）。

---

## 2. 核心数据结构

### 2.1 BufferDesc — 缓冲区描述符

`src/include/storage/buf_internals.h:326`，每个缓冲槽一个：

```c
typedef struct BufferDesc {
    BufferTag        tag;                  // 本槽缓存哪个页（rel,fork,block）  buf_internals.h:332
    int              buf_id;               // 槽编号（0..NBuffers-1），不变
    pg_atomic_uint64 state;                // ★refcount + usagecount + flags 打包成一个原子量  :344
    int              wait_backend_pgprocno;// 等待独占 pin 的 backend
    PgAioWaitRef     io_wref;              // AIO 进行中的等待引用
    /* content lock 等待者链表 ... */
} BufferDesc;
```

**`state` 是一个 64 位原子量**，把引用计数、使用计数、标志位打包，用 CAS 无锁更新（高并发设计的核心）：
- **refcount**：pin 计数，>0 表示有人在用，不可换出。
- **usagecount**：时钟扫描的"热度"，上限 `BM_MAX_USAGE_COUNT = 5`（`buf_internals.h:144`）。
- **flags**：`BM_DIRTY`（需回写）、`BM_VALID`（数据有效）、`BM_TAG_VALID`、`BM_IO_IN_PROGRESS`、`BM_PERMANENT`、`BM_PIN_COUNT_WAITER` 等。

### 2.2 BufferTag — 页的唯一标识

```c
typedef struct BufferTag {
    Oid         spcOid;       // 表空间
    Oid         dbOid;        // 数据库
    RelFileNumber relNumber;  // 关系物理文件号
    ForkNumber  forkNum;      // main/fsm/vm/init
    BlockNumber blockNum;     // 块号
} BufferTag;
```

`(spc,db,rel,fork,block)` 全局唯一定位一个磁盘页。

### 2.3 Buffer 哈希表与分区锁

`buf_table.c` 维护 `BufferTag → buf_id` 的共享哈希表。为降低锁竞争，按 hash 分成 **`NUM_BUFFER_PARTITIONS`（128）** 个分区，每分区一把 `BufferMappingLock`（LWLock）。查/插/删某 tag 只锁它所属分区（`buf_internals.h:245-250`）。

### 2.4 两种锁

- **buffer header spinlock**（嵌在 `state` 里，或退化路径）：保护 tag 等字段的极短临界区。
- **content lock**（每 buffer 一把 LWLock）：保护页面**内容**的并发读写（共享=读、独占=改）。访问方法在读页时持共享、改页时持独占。

注意区分：**pin** 防止换出（引用计数），**content lock** 防止内容并发冲突，二者正交。

---

## 3. 核心算法

### 3.1 ReadBuffer 流程

`ReadBuffer()`（`bufmgr.c:879`）→ `ReadBufferExtended()`（`bufmgr.c:926`）→ `BufferAlloc`/`GetVictimBuffer`：

```
1. 算 BufferTag，定位 hash 分区，加 BufferMappingLock(共享)
2. BufTableLookup(tag)
   ├─ 命中：pin 该 buffer（state 原子 +refcount），放锁，返回
   └─ 未命中：
        放共享锁 → StrategyGetBuffer 选一个 victim 槽
        若 victim 是脏页：先按 WAL 先行刷盘（见 3.4）→ 再 smgrread 读入新页
        加 BufferMappingLock(独占) 插入新 tag（处理竞争：可能别人已读入）
        标 BM_VALID，pin，返回
```

`ReadBufferMode`（如 `RBM_NORMAL`、`RBM_ZERO_AND_LOCK`、`RBM_NORMAL_NO_LOG`）控制"读入还是清零""是否预加内容锁"等。

### 3.2 时钟扫描替换（clock-sweep）

`StrategyGetBuffer()`（`freelist.c:184`）选淘汰页。算法（近似 LRU，`buf_internals.h` 注释）：

```
维护一个全局"时钟指针" nextVictimBuffer，环形扫描所有 buffer：
  对当前 buffer：
    若 refcount > 0（被 pin）→ 跳过
    若 usagecount > 0       → usagecount-- （给一次缓刑），跳过
    若 usagecount == 0      → 选中为 victim
最坏需要 BM_MAX_USAGE_COUNT+1 圈才一定选出（buf_internals.h:140）
```

每次**访问**页面会把 usagecount 提到上限（`PinBuffer` 内 `+1` 至 `BM_MAX_USAGE_COUNT`，`buf_internals.h:471`），形成"越常用越难被淘汰"。freelist（真正空闲槽链表）只在启动初期或关系被删后非空，稳态下淘汰全靠时钟扫描。

### 3.3 访问策略环（BufferAccessStrategy）

`bufmgr.h` 的 `BAS_NORMAL/BAS_BULKREAD/BAS_BULKWRITE/BAS_VACUUM`。大顺序扫描、COPY、VACUUM 若用普通策略会把整个缓冲池冲刷掉（cache pollution）。策略环让这些操作只在一个**小的循环缓冲（ring）** 内复用少量槽：

- `BAS_BULKREAD`（顺序扫描，约 256KB ring）、`BAS_VACUUM`、`BAS_BULKWRITE`（COPY/CTAS）。
- `StrategyGetBuffer` 带 strategy 时优先从 ring 取槽并复用，避免淘汰热数据。

### 3.4 WAL 先行：刷脏页前的铁律

回写一个脏页前（`FlushBuffer`），必须保证保护它的 WAL 已落盘：

```
XLogFlush(BufferGetLSN(buf));   // 确保 WAL 已 fsync 到 >= 该页 pd_lsn
smgrwrite(...);                 // 再写数据页
```

否则崩溃后数据页的修改"先于"其 WAL 持久化，恢复时无法重放该修改 → 数据不一致。`pd_lsn`（[15-page-layout](15-page-layout.md)）正是这条规则的判据。

### 3.5 检查点与脏页回写

`BufferSync()`（检查点时）扫描所有脏 buffer 排序后批量刷盘；后台 `bgwriter` 平时也预刷部分脏页，平滑 I/O 峰值（见 [../process/51-background-procs](../process/51-background-procs.md)）。

---

## 4. ReadStream — 预读框架

`src/include/storage/read_stream.h`。把"决定读哪些块"与"实际（异步）读取"解耦，给顺序/索引扫描提供高效预读：

```c
ReadStream *read_stream_begin_relation(int flags, BufferAccessStrategy strategy,
    Relation rel, ForkNumber forknum,
    ReadStreamBlockNumberCB callback, void *private, size_t per_buffer_data_size);
Buffer read_stream_next_buffer(ReadStream *stream, void **per_buffer_data);
```

调用方提供 callback"下一个要读的块号"，ReadStream 据此提前发起多个（可能异步的，见 [17-smgr-forks](17-smgr-forks.md) 的 AIO）读取，调用方 `read_stream_next_buffer` 时往往已就绪。`READ_STREAM_SEQUENTIAL` 提示顺序访问；`READ_STREAM_MAINTENANCE` 用 `maintenance_io_concurrency`。heap 顺序扫描、VACUUM、btree 等已改用 ReadStream（[11-heap](11-heap.md)）。

---

## 5. 设计模式

- **无锁状态机（原子 state）**：把 refcount/usagecount/flags 打进一个 64 位原子量，用 CAS 更新，避免高频路径上的锁，是高并发缓冲池的核心。
- **分区锁降竞争**：哈希表按 tag 分 128 区，每区独立 LWLock，把全局锁热点打散。
- **时钟扫描近似 LRU**：用 usagecount + 环形指针以 O(1) 摊销代价近似 LRU，避免维护精确 LRU 链的开销与锁争用。
- **正交的 pin 与 content lock**：换出保护与内容并发保护分离，让"读页内容"与"页是否可被替换"互不耦合。
- **策略环防污染**：大扫描限定在小 ring 内复用槽，保护工作集——把"扫描模式"作为一等参数下传。
- **预读解耦（ReadStream）**：调用方只说"接下来读哪些块"，框架负责批量/异步发起，把 I/O 延迟隐藏在计算之后。

---

## 6. 架构编排

```
访问方法（heap_getnextslot / _bt_search）
  │ ReadBuffer / ReadBufferExtended / read_stream_next_buffer
  ▼  bufmgr.c
查 BufTable（128 分区哈希，BufferMappingLock）
  命中 → pin → 返回（持 content lock 读/写）
  未命中 → StrategyGetBuffer（时钟扫描/策略环）  freelist.c:184
         → 脏 victim：XLogFlush(WAL先行) → smgrwrite
         → smgrread 读入                          ▼ 见 17-smgr-forks
回写：MarkBufferDirty → 检查点/bgwriter → FlushBuffer（再 WAL 先行）→ smgrwrite
```

---

## 7. 动手探索

```sql
SHOW shared_buffers;

CREATE EXTENSION pg_buffercache;
-- 缓冲池里有哪些关系的页、有多少脏页、usagecount 分布
SELECT c.relname, count(*) AS buffers,
       count(*) FILTER (WHERE b.isdirty) AS dirty,
       round(avg(b.usagecount),1) AS avg_usage
FROM pg_buffercache b JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname ORDER BY buffers DESC LIMIT 10;

-- 命中率与块读
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t WHERE id < 1000;  -- shared hit/read
```

调试：`p *GetBufferDescriptor(buf-1)` 看某 buffer 的 tag/state；`pg_atomic_read_u64(&desc->state)` 解析 refcount/usagecount/flags。

---

## 相关模块

- 下层磁盘：[17-smgr-forks](17-smgr-forks.md)
- 页面内容：[15-page-layout](15-page-layout.md)
- WAL 先行：[../transaction/33-wal](../transaction/33-wal.md)
- 锁：[../concurrency/21-lwlock](../concurrency/21-lwlock.md)
- 后台刷盘：[../process/51-background-procs](../process/51-background-procs.md)
