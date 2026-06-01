# 共享内存与 IPC

> 源码：`src/backend/storage/ipc/`（`shmem.c` `ipc.c` `procsignal.c` `sinvaladt.c` `dsm.c` `shm_mq.c` `shm_toc.c`）
> 头文件：`src/include/storage/{shmem,dsm,shm_mq,shm_toc}.h`

---

## 1. 职责

多进程架构下，进程间共享状态全靠**共享内存**，协作靠各种 **IPC** 原语。本模块提供：
- **主共享内存段**：启动时分配的大块共享内存，容纳缓冲池、锁表、PGPROC 数组、WAL buffer 等。
- **动态共享内存（DSM）**：运行期按需分配/释放的共享段，主要给并行查询用。
- **进程间消息/信令**：共享消息队列（`shm_mq`）、进程信号（`procsignal`）、共享失效队列（`sinvaladt`）。

---

## 2. 主共享内存段

### 2.1 分配

postmaster 启动时 `CreateSharedMemoryAndSemaphores()`（`ipci.c`）一次性算出所有子系统的共享内存需求（每个子系统报 `XXXShmemSize()`），分配一整块（Unix 下 `mmap` 匿名共享映射 + `shmget` 兜底），各 backend fork 时自动继承该映射（Windows `EXEC_BACKEND` 则重新附加）。

### 2.2 ShmemAlloc 与 ShmemIndex

`shmem.c`：主段内用简单的 bump 分配器 `ShmemAlloc()` 切分。各子系统在初始化时调 `ShmemInitStruct("name", size, &found)` 拿到自己那块（按名字登记在 `ShmemIndex` 哈希表，保证各进程看到同一地址）。**主段大小固定、启动后不变**——所以 `shared_buffers`、`max_connections` 等影响共享内存的参数改动需重启。

### 2.3 主要住户

| 结构 | 子系统 | 文档 |
|------|--------|------|
| Buffer Pool + BufferDesc + BufTable | 缓冲管理 | [../storage/16](../storage/16-buffer-manager.md) |
| WAL Buffers + XLogCtl | WAL | [../transaction/33](../transaction/33-wal.md) |
| LWLock 数组、锁表（LOCK/PROCLOCK） | 锁 | [../concurrency/21](../concurrency/21-lwlock.md), [22](../concurrency/22-heavyweight-lock.md) |
| PGPROC 数组 + ProcArray | 进程/快照 | [../concurrency/20](../concurrency/20-lock-overview.md), [../transaction/32](../transaction/32-mvcc-snapshot.md) |
| CLOG/SUBTRANS/MultiXact SLRU buffers | 事务状态 | [../transaction/31](../transaction/31-clog-slru.md) |
| 共享失效队列（SI） | 缓存一致性 | [../catalog/41](../catalog/41-caches.md) |
| 统计（PG15+ 共享内存统计） | pgstat | — |

---

## 3. 进程信令

### 3.1 Latch

最基础的睡眠/唤醒原语，见 [../concurrency/20-lock-overview](../concurrency/20-lock-overview.md) §3。`WaitLatch`/`SetLatch`，跨进程、信号安全。

### 3.2 ProcSignal

`procsignal.c`：让一个进程给另一个进程发"请处理某类事件"的异步通知（如 `PROCSIG_RECOVERY_CONFLICT_*`、`PROCSIG_BARRIER`）。通过共享内存标志 + `SIGUSR1` 实现：目标进程在中断检查点（`CHECK_FOR_INTERRUPTS()`）处理。

### 3.3 共享失效队列（sinvaladt）

`sinvaladt.c`：环形队列广播缓存失效消息，是 [../catalog/41-caches](../catalog/41-caches.md) 的传输层。每个 backend 有读游标，落后太多则触发 reset。

---

## 4. 动态共享内存（DSM）

`dsm.c`，`src/include/storage/dsm.h`：

```c
dsm_segment *dsm_create(Size size, int flags);   // 建一个新 DSM 段       dsm.h:34
dsm_segment *dsm_attach(dsm_handle h);            // 用 handle 附加已有段   dsm.h:35
void        *dsm_segment_address(dsm_segment *);
void         dsm_detach(dsm_segment *);
```

主段固定大小不够灵活，DSM 提供**运行期按需**的共享段：创建者拿 `dsm_handle`（一个整数），把 handle 传给别的进程（经主共享内存），后者 `dsm_attach`。主要用户是**并行查询**（见 [53-parallel-query](53-parallel-query.md)）。后端实现可选 `posix`（`shm_open`）、`sysv`、`mmap` 等（`dynamic_shared_memory_type`）。

### 4.1 shm_toc —— 段内目录

`shm_toc.c`：一个 DSM 段里要放多样东西（计划、参数、各种状态），`shm_toc`（table of contents）在段内建一个"key → 偏移"的目录，让附加方按 key 找到各部分，类似段内的 `ShmemIndex`。

### 4.2 shm_mq —— 共享消息队列

`shm_mq.c`，`shm_mq.h:52`：在共享内存上实现的**单生产者单消费者环形消息队列**，支持阻塞/非阻塞、用 latch 通知。并行 worker 用它把结果元组流回 leader（Gather）、把错误回传给 leader（`ParallelContext.error_mqh`，见 [53](53-parallel-query.md)）。

---

## 5. 设计模式

- **固定主段 + 命名登记（ShmemInitStruct）**：启动时一次性按各子系统申报量分配，用名字哈希保证各进程看到同一结构同一地址——多进程共享状态的统一接入点。
- **DSM + handle 传递**：运行期共享用"建段拿整数 handle、别人凭 handle 附加"解耦生命周期，弥补主段固定大小的不足。
- **段内目录（shm_toc）**：在一块裸共享内存里用 key→偏移目录组织多个对象，使附加方无需预知布局。
- **SPSC 环形队列（shm_mq）+ latch**：用无锁环形缓冲 + latch 通知实现高效跨进程流式传输，承载并行结果与错误回传。
- **异步信令（ProcSignal + CHECK_FOR_INTERRUPTS）**：把"请你处理某事"投递为共享标志 + 信号，目标在安全点（中断检查）响应，避免在任意点被打断的复杂性。

---

## 6. 架构编排

```
启动：postmaster → CreateSharedMemoryAndSemaphores
        └─ 各子系统 ShmemInitStruct 切分主段（buffer/锁/PGPROC/WAL/SLRU/SI…）
fork：backend 继承主段映射（EXEC_BACKEND 则重新附加）

运行期协作：
  唤醒：SetLatch / WaitLatch
  异步事件：ProcSignal + SIGUSR1 → CHECK_FOR_INTERRUPTS 处理
  缓存一致性：sinvaladt 环形队列广播失效（见 catalog/41）
并行查询：
  leader：dsm_create → shm_toc 写入计划/参数 → 启 worker（传 dsm_handle）
  worker：dsm_attach → shm_toc 按 key 取 → shm_mq 把元组/错误流回 leader（见 53）
```

---

## 7. 动手探索

```sql
SHOW shared_buffers;                          -- 主段最大头
SHOW dynamic_shared_memory_type;              -- DSM 后端
SELECT name, allocated_size FROM pg_shmem_allocations ORDER BY allocated_size DESC LIMIT 15;
SELECT * FROM pg_shmem_allocations WHERE name LIKE '%Buffer%';
```

```bash
ipcs -m    # 查看 System V 共享内存段（若用 sysv）
ls /dev/shm/    # POSIX 共享内存（DSM posix / mmap 模式可见）
```

---

## 相关模块

- 分配者：[50-postmaster](50-postmaster.md)
- 主要住户们：[../storage/16](../storage/16-buffer-manager.md)、[../concurrency/20](../concurrency/20-lock-overview.md)、[../transaction/33](../transaction/33-wal.md)
- DSM 主用户：[53-parallel-query](53-parallel-query.md)
- 失效队列消费者：[../catalog/41-caches](../catalog/41-caches.md)
