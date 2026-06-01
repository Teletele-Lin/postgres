# MemoryContext — 基于区域的内存管理

> 源码：`src/backend/utils/mmgr/`（`mcxt.c` `aset.c` `generation.c` `slab.c`）
> 头文件：`src/include/nodes/memnodes.h`、`src/include/utils/memutils.h`

---

## 1. 职责

PostgreSQL **不直接用 `malloc/free`**，而是用层级化的 **MemoryContext**（内存上下文）管理几乎所有动态内存。核心理念：把内存按"生命周期"分组到上下文里，一次性 **reset/delete 整个上下文** 批量释放，而非逐对象 free。

好处：
- **防泄漏**：异常（`ereport(ERROR)`）发生时无需逐个 free——直接把事务/查询的上下文整体重置，所有分配自动回收。
- **高效**：executor 每处理一行就 reset per-tuple 上下文，O(1) 释放该行所有临时内存。
- **可诊断**：上下文带名字，能打印内存用量定位泄漏。

这是贯穿全内核的内存范式（见 [../00-overview](../00-overview.md) §7）。

---

## 2. 核心数据结构

### 2.1 MemoryContextData + 方法虚表

`src/include/nodes/memnodes.h`。上下文是一个对象，行为由 `MemoryContextMethods`（`memnodes.h:58`）函数指针虚表分发：

```c
typedef struct MemoryContextMethods {
    void *(*alloc)(MemoryContext, Size, int flags);     // 分配               memnodes.h:66
    void  (*free_p)(void *pointer);                     // 释放单个（很少用）
    void *(*realloc)(void *pointer, Size, int flags);   // 改大小             memnodes.h:76
    void  (*reset)(MemoryContext);                      // 释放本上下文所有分配 memnodes.h:83
    void  (*delete_context)(MemoryContext);             // 销毁上下文
    Size  (*get_chunk_space)(void *pointer);
    ...
} MemoryContextMethods;

typedef struct MemoryContextData {
    NodeTag       type;
    MemoryContextMethods *methods;   // 虚表（决定是 AllocSet/Generation/Slab）
    MemoryContext parent;            // 父上下文
    MemoryContext firstchild;        // 子上下文链
    MemoryContext nextchild;
    const char   *name;              // 名字（诊断用）
    ...
} MemoryContextData;
```

**层级树**：每个上下文有父、有子。删除一个上下文会**递归删除所有子上下文**——这是"reset 事务上下文即清掉事务内一切分配"的关键。

### 2.2 三种实现（不同分配模式）

| 实现 | 文件 | 适用 |
|------|------|------|
| **AllocSet** | `aset.c` | 默认通用分配器；按 2 的幂大小分桶 freelist + 向 malloc 批量要大块再切分 |
| **Generation** | `generation.c` | 分配后大多同时释放的场景（如 ReorderBuffer）；按代回收，碎片少 |
| **Slab** | `slab.c` | 大量**等大**对象（如固定大小的元组）；定长块，O(1) 分配释放 |

`AllocSetContextCreate()` 是最常用入口。三者共享上述虚表接口，调用方对实现无感。

---

## 3. 标准上下文层级

`mcxt.c` 在启动时建立顶层，各子系统挂在下面：

```
TopMemoryContext（进程级，永生）
 ├─ ErrorContext（错误处理专用，永远预留以保证报错时有内存）
 ├─ CacheMemoryContext（relcache/catcache 等长生命周期缓存）
 ├─ MessageContext（当前客户端消息/查询的解析等）
 ├─ TopTransactionContext（当前事务，提交/回滚时重置）
 │    └─ CurTransactionContext / 各种查询、执行上下文
 │         └─ ExprContext.ecxt_per_tuple_memory（每元组，处理一行就 reset）
 └─ PortalContext（游标/portal 生命周期）
```

`CurrentMemoryContext` 是全局变量，`palloc()` 默认从它分配。切换上下文用 `MemoryContextSwitchTo(ctx)`（返回旧的，配对恢复）。

---

## 4. 核心用法与算法

### 4.1 palloc / pfree

```c
old = MemoryContextSwitchTo(some_ctx);
ptr = palloc(size);       // 从 CurrentMemoryContext 分配
...
MemoryContextSwitchTo(old);
// 通常不 pfree(ptr)；等 some_ctx 被 reset/delete 时统一回收
```

`palloc` 失败（OOM）直接 `ereport(ERROR)`，不返回 NULL——调用方无需检查空指针。

### 4.2 生命周期对齐释放

不同上下文对齐不同生命周期，到点整体回收：
- **每行**：`ResetExprContext()` 重置 per-tuple 上下文（执行器每处理一行，见 [../query/06-executor-overview](../query/06-executor-overview.md)）。
- **每条命令/消息**：`MessageContext` 在读下一条客户端消息前重置。
- **每个事务**：`TopTransactionContext` 在提交/回滚时删除（见 [../transaction/30-xact](../transaction/30-xact.md)）——这也是回滚能"自动清干净"的原因。

### 4.3 AllocSet 的分配策略

`aset.c`：小块按大小 round 到 2 的幂，从对应 freelist 取/还；向 OS 批量 `malloc` 大块（block）再切 chunk，block 大小随上下文增长翻倍（amortize malloc 开销）。reset 时把 block 还给 freelist（保留首块复用），delete 时全部 free。大块（> `allocChunkLimit`）单独 malloc，单独管理。

---

## 5. 设计模式

- **区域内存管理（region/arena）**：按生命周期分组、整体回收，把"何时释放"从"每个对象"提升到"每个上下文"，根除逐对象 free 的泄漏与开销——内核内存管理的总纲。
- **生命周期即层级树**：父子上下文映射"包含关系"，删父即删整棵子树，让"清理一个事务/查询"成为一次操作。
- **策略模式（三种实现共享虚表）**：AllocSet/Generation/Slab 按分配模式各擅胜场，调用方经统一 `palloc` 接口无感切换。
- **异常安全的天然保障**：`ereport(ERROR)` longjmp 回事务边界后重置上下文，配合 [71-error-elog](71-error-elog.md) 实现"出错即自动回收"，无需 try/finally 逐个释放。
- **OOM 即报错**：`palloc` 失败直接抛错，免去满世界的 NULL 检查，简化调用代码。

---

## 6. 架构编排

```
TopMemoryContext（进程永生）
  ├─ ErrorContext（报错时备用内存）        见 71-error-elog
  ├─ CacheMemoryContext                    见 catalog/41（relcache/catcache）
  ├─ MessageContext  ──每条命令前 reset
  └─ TopTransactionContext ──提交/回滚 delete   见 transaction/30
       └─ 查询/执行上下文
            └─ ExprContext.per_tuple ──每行 ResetExprContext   见 query/06

palloc → CurrentMemoryContext.methods->alloc（AllocSet/Generation/Slab）
回收 → MemoryContextReset/Delete（整体，递归子上下文）
```

---

## 7. 动手探索

```sql
-- 某 backend 的内存上下文用量（PG14+）
SELECT name, parent, total_bytes, used_bytes, free_bytes
FROM pg_backend_memory_contexts ORDER BY total_bytes DESC LIMIT 15;

-- 远程查看另一进程（PG14+）
SELECT * FROM pg_log_backend_memory_contexts(<pid>);  -- 输出到日志
```

调试：`call MemoryContextStats(TopMemoryContext)`（gdb）打印整棵上下文树用量，定位泄漏；`p CurrentMemoryContext->name` 看当前在哪个上下文分配。

---

## 相关模块

- 异常清理伙伴：[71-error-elog](71-error-elog.md)
- 执行器每元组重置：[../query/06-executor-overview](../query/06-executor-overview.md)
- 事务上下文：[../transaction/30-xact](../transaction/30-xact.md)
- 缓存上下文：[../catalog/41-caches](../catalog/41-caches.md)
