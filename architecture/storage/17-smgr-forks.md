# SMGR / Forks / FSM / VM / AIO — 物理存储管理

> 源码：`src/backend/storage/smgr/`（`smgr.c` `md.c`）、`storage/freespace/`、`access/heap/visibilitymap.c`、`storage/aio/`
> 头文件：`src/include/storage/smgr.h`、`src/include/common/relpath.h`

本层在 Buffer Manager 之下，把"读写某关系某 fork 某块"翻译成对操作系统文件的实际 I/O。

---

## 1. 职责

- **SMGR（Storage Manager）**：关系物理存储的抽象接口，屏蔽底层介质。当前唯一实现是 `md.c`（magnetic disk，基于普通文件）。
- **段文件管理**：一个 fork 超过 1GB（`RELSEG_SIZE`）就分段（`<relfilenode>`, `<relfilenode>.1`, `.2`…），`md.c` 负责段的定位、扩展、截断。
- **fork 管理**：一个关系由多个 fork（main/fsm/vm/init）组成，各是独立文件。
- **AIO**：异步 I/O 子系统，让读写不阻塞 backend。

---

## 2. SMGR 接口

`src/include/storage/smgr.h`：

```c
void        smgrread(SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum, void *buffer);
void        smgrreadv(...);                                   // 向量读（多块一次）   smgr.h:100
void        smgrwrite(SMgrRelation, ForkNumber, BlockNumber, const void *, bool skipFsync);
void        smgrwritev(...);                                  // 向量写             smgr.h:107
void        smgrextend(SMgrRelation, ForkNumber, BlockNumber, const void *, bool skipFsync); // 扩展新块  smgr.h:92
BlockNumber smgrnblocks(SMgrRelation, ForkNumber);            // fork 当前块数       smgr.h:113
void        smgrtruncate(SMgrRelation, ForkNumber *, int nforks, ...);  // 截断       smgr.h:115
void        smgrwriteback(...);                              // 提示 OS 回写
```

`SMgrRelationData`（`smgr.h:35`）是打开的物理关系句柄，缓存各 fork 的文件描述符与块数。它与 relcache 的 `RelationData` 分离：smgr 句柄可在事务边界被清理而 relcache 仍在。

### 向量 I/O 与 AIO 接口

`smgrreadv/smgrwritev` 接收 buffer 数组，一次系统调用读写多个连续块，配合 ReadStream 的预读。AIO 路径下读写以非阻塞方式提交（见 §5）。

---

## 3. fork —— 一个关系的多个物理文件

`src/include/common/relpath.h:59`：

```c
typedef enum ForkNumber {
    MAIN_FORKNUM = 0,        // 表/索引的实际数据
    FSM_FORKNUM,             // Free Space Map
    VISIBILITYMAP_FORKNUM,   // Visibility Map（仅 heap）
    INIT_FORKNUM,            // unlogged 表的初始化 fork
} ForkNumber;
```

磁盘上：main fork = `base/<db>/<relfilenode>`，FSM = `<relfilenode>_fsm`，VM = `<relfilenode>_vm`，init = `<relfilenode>_init`。

### 3.1 FSM — Free Space Map（`freespace/`）

记录每个数据页**大致剩余空闲空间**（每页用 1 字节量化为 0–255 级）。组织成一棵页内三层 + 跨页的树，`fsm_search` 能快速找到"至少有 N 字节空闲"的页。`heap_insert` 的 `RelationGetBufferForTuple` 靠它避免逐页扫描找放置位置（见 [11-heap](11-heap.md)）。FSM 是**近似**的、不写 WAL（崩溃后可能不准，但只影响空间利用不影响正确性，VACUUM 会重建）。

### 3.2 VM — Visibility Map（`visibilitymap.c`）

每个数据页 **2 bit**：
- `ALL_VISIBLE`：该页所有元组对所有事务可见 → **Index-Only Scan 可跳过回表**；VACUUM 可跳过该页。
- `ALL_FROZEN`：该页所有元组已冻结 → anti-wraparound VACUUM 可跳过。

VM 的正确性关乎可见性，**写 WAL**。VACUUM 设置位，任何对页的修改清除位。VM 极小（10亿行表的 VM 仅几 MB），是 index-only scan 与高效 VACUUM 的关键。

### 3.3 init fork

`unlogged` 表/索引的"空内容模板"。崩溃恢复时，unlogged 关系的 main fork 被 init fork 覆盖（清空），因为 unlogged 数据不写 WAL、崩溃后不保证一致，干脆重置为空。

---

## 4. md.c —— 基于文件的实现

`md.c` 把 SMGR 调用映射到 `FileRead`/`FileWrite`（经 `fd.c` 的虚拟文件描述符池 `VFD`，复用有限的 OS fd）：

- **段拆分**：块号 / `RELSEG_SIZE` 得段号，定位到 `<relfilenode>.<segno>` 文件内偏移。
- **延迟 fsync**：写时 `skipFsync` 常为真，把"需要 fsync 的文件"通过 `register_dirty_segment` 交给 **checkpointer** 在检查点时统一 fsync（fsync 吸收，`sync.c`），避免每次写都 fsync。
- **扩展**：`smgrextend` 在 fork 末尾写新块，必要时新建段文件。

`fd.c` 的 VFD 机制：进程能打开的 OS 文件数有限，VFD 池按 LRU 复用，逻辑上可"同时打开"远超 OS 限制的文件。

---

## 5. AIO — 异步 I/O（`storage/aio/`）

PostgreSQL 较新的异步 I/O 子系统，让 backend 提交 I/O 后不阻塞、继续干活，I/O 完成再处理结果。要点：

- **后端模式**（`io_method`）：`sync`（同步，退化）、`worker`（专门的 io worker 进程代为阻塞 I/O）、`io_uring`（Linux 原生异步）。
- **接口**：`pgaio_io_acquire` 取一个 I/O 句柄，提交读/写，`pgaio_io_wait` 或在需要数据时等待完成。
- **与 ReadStream 协同**：ReadStream（[16-buffer-manager](16-buffer-manager.md)）通过 AIO 提前发起多个块的读，把磁盘延迟与上层计算重叠，大幅提升顺序扫描/VACUUM 吞吐。

---

## 6. 设计模式

- **存储抽象（SMGR）**：关系物理存储面向接口（`f_smgr` 函数表），当前虽只有 md.c，但为未来介质（如直接管理裸设备）预留扩展点。
- **多 fork 分离关注点**：数据、空闲空间、可见性、unlogged 模板各用独立文件，互不干扰，可独立重建/截断。
- **正确性分级（FSM 不写 WAL，VM 写 WAL）**：只影响性能的元数据（FSM）放弃崩溃一致性以省开销；影响正确性的元数据（VM）严格 WAL 保护——按"错了会怎样"决定持久化强度。
- **fsync 吸收（延迟到检查点）**：把高频的 per-write fsync 合并到检查点统一做，用 checkpointer 集中承担持久化点。
- **VFD 池**：用 LRU 复用稀缺的 OS 文件描述符，对上层呈现"无限文件"假象。
- **异步重叠（AIO + ReadStream）**：以"提交-继续-完成"模型把 I/O 等待隐藏在计算后。

---

## 7. 架构编排

```
Buffer Manager（FlushBuffer / ReadBuffer / ReadStream）
  │ smgrread / smgrwrite / smgrextend / smgrreadv
  ▼  storage/smgr/smgr.c（f_smgr 分发）
md.c（magnetic disk）
  ├─ 块号 → 段文件 + 段内偏移（RELSEG_SIZE 拆段）
  ├─ fd.c VFD 池 → OS 文件描述符
  ├─ register_dirty_segment → checkpointer 延迟 fsync（sync.c）
  └─ AIO：pgaio 提交/等待（io_method = worker / io_uring）
  ▼
OS 文件：base/<db>/<relfilenode>[.segno] [_fsm|_vm|_init]
```

---

## 8. 动手探索

```sql
-- 关系的物理文件位置与各 fork
SELECT pg_relation_filepath('t');                 -- base/<db>/<relfilenode>
SELECT pg_relation_size('t', 'main'),
       pg_relation_size('t', 'fsm'),
       pg_relation_size('t', 'vm');

-- VM / FSM 内容
CREATE EXTENSION pg_visibility;  CREATE EXTENSION pg_freespacemap;
SELECT * FROM pg_visibility_map('t'::regclass);   -- all_visible/all_frozen
SELECT blkno, avail FROM pg_freespace('t');       -- 各页可用空间

-- AIO 配置
SHOW io_method;  SHOW effective_io_concurrency;  SHOW maintenance_io_concurrency;
```

```bash
ls -l $PGDATA/base/<dboid>/<relfilenode>*   # 看段文件与 _fsm/_vm
```

---

## 相关模块

- 上层缓冲：[16-buffer-manager](16-buffer-manager.md)
- FSM 的使用者：[11-heap](11-heap.md)
- VM 与可见性/VACUUM：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
- 检查点 fsync：[../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md)
