# 缓存层 — RelCache / SysCache / 失效消息

> 源码：`src/backend/utils/cache/`（`relcache.c` `catcache.c` `syscache.c` `lsyscache.c` `inval.c` `plancache.c`）
> 上游：[40-catalog](40-catalog.md)

---

## 1. 职责

系统表被**极高频**访问：每次解析列名、每次类型检查、每次打开表都要查 `pg_class`/`pg_attribute`/`pg_type`/`pg_proc`。若每次都扫系统表 + 走 MVCC，代价无法承受。本模块在每个 backend 的私有内存里缓存系统表内容，并通过**共享失效消息**在多进程间保持一致。

两个主要缓存：
- **SysCache / CatCache**：缓存单条系统表元组（按某 key）。
- **RelCache**：缓存"打开的关系"的完整描述符 `RelationData`（聚合了多张系统表的信息）。

---

## 2. CatCache / SysCache —— 系统表元组缓存

### 2.1 结构

`catcache.c` 实现按 key 哈希的元组缓存（`CatCache`）。`syscache.c` 在其上封装一组**具名缓存**（`SysCacheIdentifier`，如 `RELOID`、`ATTNAME`、`PROCOID`、`TYPEOID`），每个对应"某系统表 + 某唯一索引"。

```c
HeapTuple SearchSysCache1(int cacheId, Datum key1);   // syscache.c:221
// 例：SearchSysCache1(TYPEOID, ObjectIdGetDatum(typeoid)) → pg_type 那一行
ReleaseSysCache(tuple);
```

`SearchSysCache`（`syscache.c:209`）命中则返回缓存元组；未命中则用对应索引扫系统表，把结果缓存再返回。`lsyscache.c` 提供更友好的便捷函数（`get_rel_name`、`get_func_rettype`、`get_opcode`…），内部都走 syscache。

### 2.2 负缓存

CatCache 还缓存**查不到**的结果（negative entry），避免对"不存在的对象"反复扫表——常见于名称解析尝试多个 schema。

---

## 3. RelCache —— 关系描述符缓存

### 3.1 RelationData

`src/include/utils/rel.h` 的 `RelationData`（即 `Relation`）是"打开一张表/索引"的运行期句柄，聚合了来自多张系统表的信息：

```c
typedef struct RelationData {
    RelFileLocator rd_locator;     // 物理文件定位
    Form_pg_class  rd_rel;         // pg_class 那一行
    TupleDesc      rd_att;         // 列描述符（来自 pg_attribute）
    List          *rd_indexlist;   // 该表的索引 OID 列表
    Bitmapset     *rd_indexattr;   // 被索引的列
    TriggerDesc   *trigdesc;       // 触发器
    RuleLock      *rd_rules;       // 规则（见 query/03-rewriter）
    struct TableAmRoutine *rd_tableam;  // 表访问方法虚表（见 storage/10）
    PartitionKey   rd_partkey;     // 分区键
    ...
} RelationData;
```

`RelationIdGetRelation(oid)`（`relcache.c:2089`）按 OID 取关系：命中 RelCache 直接返回，未命中调 `RelationBuildDesc()`（`relcache.c:1055`）扫多张系统表组装一个 `RelationData` 并缓存。`table_open()`/`index_open()` 在此之上加锁。

### 3.2 引用计数

RelCache 条目带引用计数（pin）：`RelationIdGetRelation` +1，`RelationClose` -1。事务结束时检查无泄漏。被失效但仍被引用的条目延迟到引用归零才重建。

---

## 4. 缓存一致性：失效消息（inval.c）

多进程各有私有缓存，一个 backend 改了系统表（如 `ALTER TABLE`），其他 backend 的缓存就过期了。解决靠 **共享失效消息（shared invalidation, SI）**：

### 4.1 机制

`inval.c` + `sinvaladt.c`（共享内存的 SI 消息环形队列）：

```
backend A：ALTER TABLE t ADD COLUMN ...
  └─ 修改 pg_class/pg_attribute（事务内）
  └─ 注册失效消息（RegisterRelcacheInvalidation 等）
  └─ COMMIT 时：把失效消息广播到共享 SI 队列
其他 backend：
  └─ 在事务边界 / 加锁时 AcceptInvalidationMessages
  └─ 读取 SI 队列中的新消息 → 使本地对应缓存条目失效（下次访问重建）
```

### 4.2 消息类型

`src/include/storage/sinval.h`：
- **CatCache inval**：使某系统表缓存的某条（或某 hash 桶）失效。
- **RelCache inval**：使某关系的 `RelationData` 失效（下次重建）。
- **SMGR inval**：物理文件变更（如 TRUNCATE 换了 relfilenode）。
- **Snapshot inval**：快照相关失效。
- **Relmap inval**：关系映射文件变更（nailed 系统表的物理位置）。

### 4.3 关键时机

失效在**提交时**广播（保证别人看到的是已提交的新元数据）；接收在**事务开始/获取锁**时（`AcceptInvalidationMessages`）。这与 [../concurrency/22-heavyweight-lock](../concurrency/22-heavyweight-lock.md) 配合：拿到表锁时必然处理完失效，看到最新定义。SI 队列满时会发"reset"消息让 backend 整体清缓存（overflow 保护）。

---

## 5. 计划缓存（plancache.c）

`plancache.c` 缓存 prepared statement 的解析树与执行计划（`CachedPlanSource`/`CachedPlan`）。它也订阅失效消息：若计划依赖的表/函数被 DDL 改了，缓存计划失效、下次重新规划。还实现 **generic plan vs custom plan** 的抉择——前几次用绑定参数定制规划，若 generic plan（参数无关）代价相当则切换到 generic 省去重复规划（见 [../query/00 流水线](../00-overview.md) §3.3）。

---

## 6. 设计模式

- **私有缓存 + 共享失效（最终一致）**：每进程缓存读多写少的元数据以求速度，靠提交时广播、加锁时接收的失效消息保证一致——多进程架构下的标准缓存一致性方案。
- **聚合缓存（RelCache）**：把分散在多张系统表的关系信息组装成一个即用的 `RelationData`，避免每次打开表都重扫多表。
- **负缓存**：缓存"不存在"的结论，挡住对缺失对象的反复查表（名称解析多 schema 尝试的关键优化）。
- **引用计数延迟重建**：被失效但在用的条目延迟到引用归零再重建，避免拔掉正在使用的描述符。
- **失效消息的类型化**：按 catcache/relcache/smgr/snapshot 分类失效，精确使最小范围的缓存过期。
- **依赖驱动的计划失效（plancache）**：缓存计划订阅其依赖对象的失效，把"何时该重新规划"交给同一套失效总线。

---

## 7. 架构编排

```
读路径：
  名称解析/类型检查/打开表
    ├─ SearchSysCache（catcache/syscache）          syscache.c:209
    │     命中 → 返回；未命中 → 扫系统表（用唯一索引）→ 缓存
    └─ RelationIdGetRelation（relcache）             relcache.c:2089
          命中 → 返回 RelationData；未命中 → RelationBuildDesc 扫多表组装  relcache.c:1055
写路径（DDL）：
  修改系统表 → RegisterInvalidation → COMMIT 广播到共享 SI 队列            inval.c
其他 backend：
  事务开始/加锁 → AcceptInvalidationMessages → 失效本地 catcache/relcache/plancache 条目
```

---

## 8. 动手探索

```sql
-- 缓存命中省去系统表扫描：重复打开同一表第二次更快（难直接观测，可借 perf）

-- DDL 后其他会话自动看到新定义（失效消息生效）
-- 会话A: ALTER TABLE t ADD COLUMN c int;
-- 会话B: SELECT c FROM t;   -- 立即可见（B 在加锁时处理了失效）

-- 计划缓存：generic vs custom plan
PREPARE p AS SELECT * FROM t WHERE id=$1;
EXPLAIN EXECUTE p(1);   -- 多次执行后可能从 custom 切到 generic plan
SELECT name, generic_plans, custom_plans FROM pg_prepared_statements;
```

调试：`relcache.c` 的 `RelationCacheInvalidate`、`inval.c` 的 `LocalExecuteInvalidationMessage` 是观察失效流的好断点。

---

## 相关模块

- 数据来源：[40-catalog](40-catalog.md)
- 名称解析使用者：[../query/02-analyzer](../query/02-analyzer.md)
- AM 虚表存放处：[../storage/10-table-am](../storage/10-table-am.md)
- 失效广播与锁配合：[../concurrency/22-heavyweight-lock](../concurrency/22-heavyweight-lock.md)
