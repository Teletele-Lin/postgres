# 谓词锁与 SSI（可串行化快照隔离）

> 源码：`src/backend/storage/lmgr/predicate.c`
> **必读** 设计文档：`src/backend/storage/lmgr/README-SSI`
> 上层总览：[20-lock-overview](20-lock-overview.md)

---

## 1. 职责

实现 `SERIALIZABLE` 隔离级别，让并发事务的执行结果**等价于某种串行执行**，而又**不真正串行化、不加阻塞读锁**。PostgreSQL 采用 **SSI（Serializable Snapshot Isolation，可串行化快照隔离）**：在 REPEATABLE READ 的快照隔离之上，追踪事务间的**读写冲突**，发现可能破坏可串行性的"危险结构"时主动回滚某个事务。

谓词锁（predicate lock，内部叫 **SIREAD lock**）是 SSI 用来记录"某事务读了什么"的非阻塞标记。

---

## 2. 背景：快照隔离的漏洞

REPEATABLE READ（快照隔离）能挡住脏读、不可重复读、幻读，但**不能**完全可串行化——存在 **write skew（写偏斜）** 异常。经典例子：两个事务各自读取一组数据、基于读到的内容写入互不相同的行，单独看都满足约束，但合在一起违反了全局不变量（如"值班表至少一人在岗"，两人同时申请离岗，各自看到对方在岗就都离岗了）。

快照隔离允许这种异常，因为它只保证"写写不冲突"，不管"读写依赖"。SSI 的使命就是补上这个洞。

---

## 3. 核心理论：危险结构

`README-SSI:95,135,161-169` 给出 SSI 的理论基石。Cahill 等人证明：任何**不可串行化**的执行，其冲突图中必然存在一个"危险结构"——**两条相邻的 rw-conflict 边**：

```
      Tin ------> Tpivot ------> Tout
         (rw)          (rw)
```

- **rw-conflict（读写冲突）**（`README-SSI:135`）：事务 A 读了一份数据，并发事务 B 写了这份数据，且 B 的写"逻辑上发生在 A 的读之后"（A 没看到 B 的写）。记 `A → B`。
- **Tpivot** 是枢轴：它既被别人 rw-依赖（`Tin→Tpivot`），又 rw-依赖别人（`Tpivot→Tout`）。
- 当这种结构嵌入一个环时，就出现不可串行化异常。

SSI 的策略（`README-SSI:167-171`）：**只监视这个危险结构**，一旦出现就回滚其中一个事务。它只需追踪**并发事务间**的 rw-conflict，不需要完整冲突图。代价是**可能误杀**（false positive）——不是每个危险结构都真的在环里，但保守回滚保证了绝不放过真异常（无 false negative）。

---

## 4. 核心数据结构

### 4.1 SERIALIZABLEXACT（SXACT）

每个 serializable 事务一个，记录它的 rw-conflict 入边/出边列表、持有的谓词锁、提交序、状态（active/committed/doomed）。SSI 的冲突图节点就是它。`inConflicts`/`outConflicts` 链表构成 rw-conflict 图的边。

### 4.2 SIREAD 锁（谓词锁）

记录"某 SXACT 读取了某个范围"。**多粒度**（`README-SSI:289-296`）：元组级、页级、关系级。粒度提升（promotion）：同一事务对同页太多元组加 SIREAD 时，合并为一个页级锁（再多则升到关系级），控制内存。

关键性质（`README-SSI:281`）：**SIREAD 锁不阻塞任何人**。它不是传统意义的锁，只是一个"我读过这里"的痕迹，供写者检测冲突。这正是 SSI 不牺牲读并发的原因。

---

## 5. 核心算法

### 5.1 读：登记 SIREAD 锁

serializable 事务读取数据时（顺序扫描、索引扫描），`predicate.c` 的 `PredicateLockTuple`/`PredicateLockPage`/`PredicateLockRelation` 给读到的元组/页/关系打 SIREAD 锁，记到本事务的 SXACT。索引扫描时锁的是**索引页范围**（表达"我读了这个键区间"），从而能捕获后来插入该区间的幻影行（谓词意义上的"读"）。

优化（`README-SSI:310-313`）：若一次修改本身已产生 ww-依赖，就不必再为它的读加元组级 SIREAD（rw-conflict 会被 ww-依赖覆盖）。

### 5.2 写：检测 rw-conflict

事务**写**数据时（`README-SSI:315-320`）：

- **修改一个堆元组** → 与所有持有该元组、或其所在页、或关系 SIREAD 锁的事务产生 **rw-conflict**（写者发现"有人读过我正在改的东西"）。
- **插入新元组** → 与持有**关系级** SIREAD 锁的事务冲突（对方做过全表/范围读，可能本应看到这行——幻读）。

每检测到一条 rw-conflict，就在两个 SXACT 间连一条边。

### 5.3 提交时判定危险结构

随着边的累积，SSI 持续检查是否形成 `Tin→Tpivot→Tout` 的两条相邻 rw-conflict 边。一旦某事务成为 pivot 且满足回滚条件（考虑提交先后，如 `README-SSI:180` "若 Tout 在 Tpivot 和 Tin 之前提交"等精化条件以减少误杀），就把某个参与事务标记为 **doomed**，其下次操作或提交时报错：

```
ERROR: could not serialize access due to read/write dependencies among transactions
(SQLSTATE 40001)
```

应用应捕获 40001 并**重试整个事务**。这是 SERIALIZABLE 的使用契约：逻辑简单（无需手工加锁防 write skew），但要准备好重试。

### 5.4 安全快照与只读优化

SSI 对**只读事务**有特殊优化：只读事务不写，不可能成为 pivot 的"出边源"，某些情况下能被证明"安全"（safe snapshot），从而完全免于冲突追踪开销。`READ ONLY DEFERRABLE` 事务会等待一个保证可串行化的快照点再开始，永不被回滚。

---

## 6. 设计模式

- **检测而非阻塞（乐观可串行化）**：不像传统两阶段封锁那样阻塞读，而是放行 + 追踪 rw-conflict + 事后回滚，把可串行化做成"乐观并发控制"，保住读写并发。
- **理论制导的最小监视（危险结构）**：依据"不可串行化 ⟹ 存在两条相邻 rw-conflict 边"的定理，只盯这一个结构而非完整环检测，把判定成本压到可接受。
- **保守安全（容忍 false positive）**：宁可误杀也不漏判，换取算法简单与绝对正确；把"偶尔重试"的成本转嫁给应用。
- **非阻塞多粒度谓词锁（SIREAD）**：用不阻塞的"读痕迹" + 粒度提升，在内存可控下表达"读了哪些数据/范围"，并能捕获幻影。
- **只读/延迟快照优化**：识别不可能引发异常的事务（只读、安全快照）予以豁免，降低常见场景开销。

---

## 7. 架构编排

```
SERIALIZABLE 事务
  读：Index/Seq Scan → PredicateLockTuple/Page/Relation（SIREAD，非阻塞）   predicate.c
        └─ 痕迹记入本事务 SERIALIZABLEXACT
  写：heap_insert/update/delete → CheckTargetForConflictsIn
        └─ 与持有相应 SIREAD 的并发事务连 rw-conflict 边
  持续：检测 Tin→Tpivot→Tout 危险结构（两条相邻 rw-conflict）
        └─ 命中 → 标记 doomed
  提交/下次操作：doomed → ERROR 40001（应用重试）
只读优化：safe snapshot / READ ONLY DEFERRABLE → 免追踪
```

---

## 8. 动手探索

```sql
-- write skew：SERIALIZABLE 会拦下，REPEATABLE READ 不会
-- 会话1: BEGIN ISOLATION LEVEL SERIALIZABLE;
--         SELECT count(*) FROM duty WHERE on_call;   -- 看到 2 人
-- 会话2: BEGIN ISOLATION LEVEL SERIALIZABLE;
--         SELECT count(*) FROM duty WHERE on_call;   -- 看到 2 人
-- 会话1:  UPDATE duty SET on_call=false WHERE id=1;
-- 会话2:  UPDATE duty SET on_call=false WHERE id=2;
-- 会话1:  COMMIT;
-- 会话2:  COMMIT;   -- → ERROR 40001 could not serialize access

-- 谓词锁可见（locktype='page'/'tuple' 且 mode='SIReadLock'）
SELECT locktype, relation::regclass, page, tuple, mode
FROM pg_locks WHERE mode = 'SIReadLock';

SHOW max_pred_locks_per_transaction;   -- 谓词锁内存上限
```

---

## 相关模块

- 总览：[20-lock-overview](20-lock-overview.md)
- 快照基础：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
- 索引谓词锁（btree 支持 ampredlocks）：[../storage/13-nbtree](../storage/13-nbtree.md)
