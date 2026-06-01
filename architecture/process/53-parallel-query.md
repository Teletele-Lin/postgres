# 并行查询基础设施

> 源码：`src/backend/access/transam/parallel.c`、`src/backend/executor/{nodeGather,nodeGatherMerge}.c`、`execParallel.c`
> 头文件：`src/include/access/parallel.h`

---

## 1. 职责

让单条查询用**多个进程**协作执行，利用多核加速大扫描/聚合/join。核心挑战是在 PostgreSQL 的**多进程**模型下，把一个查询的执行上下文（计划、参数、快照、catalog 状态）从 leader **复制**到临时 fork 的 worker 进程，并把结果汇回——不能像多线程那样直接共享堆内存。

并行不是默认全开：规划器评估收益与安全性（`max_parallel_hazard`，见 [../query/04-planner-overview](../query/04-planner-overview.md) §4），数据量足够大、无 parallel-unsafe 函数、非数据修改（一般）时才生成并行计划。

---

## 2. 进程模型：leader + workers

```
        ┌─────────── leader backend ───────────┐
        │  Gather / GatherMerge 节点            │
        │     ↑ 从 shm_mq 收集元组               │
        └───────────────────────────────────────┘
              ↑ shm_mq        ↑ shm_mq        ↑ shm_mq
        ┌─────────┐     ┌─────────┐     ┌─────────┐
        │ worker0 │     │ worker1 │     │ worker2 │   ← 跑同一段 partial plan
        └─────────┘     └─────────┘     └─────────┘
              ↘            ↓            ↙
                共享的 DSM 段（计划、参数、快照、各种状态）
```

- **worker** 是 postmaster 应 leader 请求 fork 的 **background worker**（`bgworker` 机制）。
- worker 与 leader 跑**同一段 partial plan**（如并行顺序扫描），各处理表的一部分块。
- worker 通过 **shm_mq**（共享消息队列，见 [52-shmem-ipc](52-shmem-ipc.md)）把结果元组流回 leader 的 **Gather** 节点。

---

## 3. 核心数据结构与流程

### 3.1 ParallelContext

`src/include/access/parallel.h:33`，并行执行的总控：

```c
typedef struct ParallelContext {
    dsm_segment    *seg;              // DSM 段
    void           *private_memory;
    shm_toc        *toc;             // 段内目录
    int             nworkers;        // 期望 worker 数
    int             nworkers_launched;
    BackgroundWorkerHandle **worker; // worker 句柄
    shm_mq_handle  *error_mqh;       // 错误回传队列              parallel.h:30
    ...
} ParallelContext;
```

### 3.2 建立流程

`CreateParallelContext()`（`parallel.c:175`）→ `InitializeParallelDSM()`（`parallel.c:213`）→ `LaunchParallelWorkers()`（`parallel.h:69`）：

```
1. CreateParallelContext：指定 worker 入口（库名+函数名）与 nworkers
2. InitializeParallelDSM：估算并建 DSM 段，用 shm_toc 把下列状态序列化进去：
     - 序列化的 PlannedStmt / 执行计划（nodeToString，见 infra/72）
     - 绑定参数、ParamListInfo
     - 活动快照（保证 worker 与 leader 看到同一 MVCC 视图）
     - 当前库/角色、GUC 设置、combo CID、临时命名空间
     - 每 worker 一个 shm_mq（元组通道）+ 错误通道
3. LaunchParallelWorkers：请 postmaster fork nworkers 个 bgworker
4. 每个 worker：ParallelWorkerMain（parallel.h:81）
     - dsm_attach + shm_toc 取回上述状态
     - 恢复出与 leader 一致的执行环境（快照/参数/GUC/catalog）
     - 跑自己那份 partial plan，元组写入 shm_mq
5. leader 的 Gather 节点从各 shm_mq 收元组；WaitForParallelWorkersToFinish 收尾
```

### 3.3 状态复制的难点

worker 必须复刻 leader 的执行语境，否则结果不一致或出错。关键复制项：
- **快照**：worker 用 leader 的同一快照（见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)），保证可见性一致。
- **combo CID / 当前命令**：让 worker 正确判断 leader 事务自身的修改可见性。
- **GUC**：worker 继承 leader 的所有参数设置（如 `work_mem`），否则代价/行为偏差。
- **错误传播**：worker 出错时，错误通过 `error_mqh` 序列化回 leader，由 leader **重新抛出**，使并行错误对用户表现得和串行一致。

---

## 4. 执行节点

- **Gather**（`nodeGather.c`）：并行计划的汇合点。启动 worker，从各 shm_mq 轮询收元组（无序）；worker 不足时 leader 自己也参与执行 partial plan（leader participation）。
- **GatherMerge**（`nodeGatherMerge.c`）：worker 各自产出**有序**结果时，按归并保持整体有序（支撑并行 ORDER BY）。
- **Partial 节点**：`Parallel Seq Scan`（worker 间用共享的 `ParallelTableScanDesc` 原子分配块范围，见 [../storage/10-table-am](../storage/10-table-am.md)）、`Parallel Bitmap Heap Scan`、`Partial Aggregate`（见 [../query/08-executor-nodes](../query/08-executor-nodes.md) 的 AggSplit）、`Parallel Hash`（共享哈希表 + Barrier 同步）、`Parallel Append`。

典型并行聚合计划：
```
Finalize Aggregate
  └─ Gather
        └─ Partial Aggregate
              └─ Parallel Seq Scan on big
```

---

## 5. 并发与锁的特殊处理

- **lock group**：leader 与其 workers 组成一个加锁组，组内视作可共享锁（避免组内"自死锁"），死锁检测按组处理（见 [../concurrency/22-heavyweight-lock](../concurrency/22-heavyweight-lock.md) §4.2）。
- **parallel-restricted / parallel-unsafe**：函数有并行安全级别标记（`proparallel`）。unsafe 函数（写数据、依赖会话状态）出现则整条查询不能并行；restricted 函数只能在 leader 跑。
- worker 处于**只读受限模式**（一般不能写数据、不能改事务状态），保证并行的安全性。

---

## 6. 设计模式

- **状态序列化跨进程复制**：多进程无法共享堆，故把计划/参数/快照/GUC 序列化进 DSM，worker 反序列化重建一致语境——这是 PostgreSQL 并行的根本手法（复用了 [../infra/72-node-system](../infra/72-node-system.md) 的 nodeToString）。
- **leader/worker + 汇合节点（Gather）**：把"分发-并行-汇合"封成执行树里的一个节点，对上层透明，可嵌套组合。
- **partial/final 拆分**：聚合等算子拆成 worker 端 partial + leader 端 final，配合 combine/serialize（见 [../query/08](../query/08-executor-nodes.md)），让原本串行的算子可并行。
- **SPSC 队列流式回传（shm_mq）**：元组与错误都走共享消息队列，带 latch 通知，背压自然。
- **错误的透明再抛**：worker 错误序列化回 leader 重抛，使并行查询的错误语义与串行无差别。
- **加锁组**：把并行进程组当作单一加锁主体，复用既有锁/死锁机制而非另起炉灶。

---

## 7. 架构编排

```
规划：standard_planner 评估并行可行性 → 生成含 Gather + Partial 节点的计划   query/04
执行：
  ExecGather（leader）
    ├─ CreateParallelContext + InitializeParallelDSM                       parallel.c:175,213
    │     └─ shm_toc 写入：序列化计划/参数/快照/GUC + 每 worker 的 shm_mq
    ├─ LaunchParallelWorkers → postmaster fork bgworker
    │     └─ worker: ParallelWorkerMain → dsm_attach → 重建语境 → 跑 partial plan
    │           → 元组写 shm_mq；出错写 error_mqh
    └─ leader 从各 shm_mq 收元组（+ 自己参与）→ 向上输出
       WaitForParallelWorkersToFinish → 重抛 worker 错误 → DestroyParallelContext
```

---

## 8. 动手探索

```sql
-- 让并行更易触发
SET max_parallel_workers_per_gather = 4;
SET parallel_setup_cost = 0; SET parallel_tuple_cost = 0;  -- 仅测试用
SET min_parallel_table_scan_size = '8kB';

EXPLAIN (ANALYZE, VERBOSE) SELECT count(*) FROM big;
-- 看到 Gather → Partial Aggregate → Parallel Seq Scan
-- Workers Launched: N

SHOW max_parallel_workers;            -- 全局 worker 上限
SELECT proname, proparallel FROM pg_proc WHERE proname='now';  -- s/r/u 安全级别
```

---

## 相关模块

- 规划侧：[../query/04-planner-overview](../query/04-planner-overview.md)
- 执行节点：[../query/08-executor-nodes](../query/08-executor-nodes.md)
- IPC 基础：[52-shmem-ipc](52-shmem-ipc.md)
- 计划序列化：[../infra/72-node-system](../infra/72-node-system.md)
- 快照复制：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
