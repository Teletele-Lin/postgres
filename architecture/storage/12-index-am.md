# Index Access Method — 索引访问方法抽象层

> 源码：`src/backend/access/index/`（通用框架）、`src/include/access/amapi.h`（虚表）
> 各实现：`nbtree/` `gin/` `gist/` `brin/` `hash/` `spgist/`
> 高层文档：`doc/src/sgml/indexam.sgml`

---

## 1. 职责

与 [Table AM](10-table-am.md) 对称，Index AM 把"索引能做什么"抽象成虚表 `IndexAmRoutine`，让 B-tree、GIN、GiST、BRIN、Hash、SP-GiST 等不同索引类型以统一接口接入规划器与执行器。它定义：建索引、插入条目、扫描（按序取 TID / 取位图）、批量删除（VACUUM）、以及一组**能力标志**告诉规划器"这种索引支持什么"。

通用框架（`index/indexam.c`、`index/genam.c`）提供 `index_open/insert/beginscan/getnext_tid/...` 包装，转发到具体 AM。

---

## 2. 核心数据结构：IndexAmRoutine

`src/include/access/amapi.h:233`。前半是**能力标志**，后半是**回调函数指针**。

### 2.1 能力标志（规划器据此选索引）

```c
typedef struct IndexAmRoutine {
    NodeTag type;
    uint16  amsupport;        // 支持函数个数（opclass 提供）
    uint16  amstrategies;     // 策略算子个数
    bool    amcanorder;       // 能产出有序结果（B-tree 是，hash/gin 否）  amapi.h:247
    bool    amcanorderbyop;   // 支持 ORDER BY 距离算子（GiST KNN）
    bool    amcanbackward;    // 支持反向扫描
    bool    amcanunique;      // 支持唯一约束
    bool    amcanmulticol;    // 支持多列
    bool    amsearcharray;    // 支持 ScalarArrayOp（IN 列表）下推         amapi.h:265
    bool    amsearchnulls;    // 支持 IS NULL 检索
    bool    amclusterable;
    bool    ampredlocks;      // 支持谓词锁（SSI 细粒度）
    bool    amcanparallel;
    bool    amcanbuildparallel;
    bool    amcaninclude;     // 支持 INCLUDE 非键列
    ...
    Oid     amkeytype;
```

`amcanorder=true` 的索引能为 ORDER BY/MergeJoin 提供 pathkeys（B-tree 独有），所以 B-tree 是最通用的索引。

### 2.2 回调函数

```c
    /* —— 构建与维护 —— */
    ambuild_function      ambuild;        // 全量建索引                amapi.h:296
    ambuildempty_function ambuildempty;   // 建空索引（unlogged 用）
    aminsert_function     aminsert;       // 插入一条                  amapi.h:298
    ambulkdelete_function ambulkdelete;   // VACUUM：批量删条目        amapi.h:300
    amvacuumcleanup_function amvacuumcleanup;

    /* —— 扫描 —— */
    ambeginscan_function  ambeginscan;                                // amapi.h:310
    amrescan_function     amrescan;
    amgettuple_function   amgettuple;     // 取下一个 TID（有序扫描）  amapi.h:312
    amgetbitmap_function  amgetbitmap;    // 一次返回全部 TID 位图      amapi.h:313
    amendscan_function    amendscan;

    /* —— 规划器接口 —— */
    amcostestimate_function amcostestimate;  // 估算索引扫描代价
    amoptions_function      amoptions;
    ...
} IndexAmRoutine;
```

`amgettuple` 与 `amgetbitmap` 二选一支持两种扫描模式：
- **amgettuple**（IndexScan）：一次取一个 TID，可保持索引序、可早停，适合 ORDER BY/LIMIT。
- **amgetbitmap**（BitmapIndexScan）：一次吐出满足条件的全部 TID 成位图，由 BitmapHeapScan 按物理页序回表，适合低选择率/多索引组合。GIN 只支持 `amgetbitmap`。

---

## 3. 算子类（opclass）—— 索引与数据类型解耦

索引 AM 不知道具体数据类型怎么比较，这通过 **operator class** 把"类型+语义"绑到索引上：
- `pg_opclass` / `pg_amop` / `pg_amproc`：定义某类型在某索引 AM 下的**策略算子**（如 B-tree 的 `<,<=,=,>=,>` 五个策略）和**支持函数**（如 B-tree 的 comparison proc）。
- 例：`int4_ops` 是 int4 在 btree 下的 opclass。`CREATE INDEX ... USING btree (col int4_ops)`。
- 这让同一索引 AM 适配任意类型（含用户自定义类型），是 PostgreSQL 索引可扩展性的核心机制。

---

## 4. 一次索引扫描的分发

```
IndexScan（执行器，nodeIndexscan.c）
  → index_getnext_tid()           index/indexam.c 包装
     → amgettuple()               虚表分发 → btgettuple (nbtree)
       → 返回 ItemPointer（TID）
  → table_index_fetch_tuple()     回 Table AM 取堆元组（见 10-table-am）
     → 可见性判断 → 输出
```

注意**职责切分**：索引 AM 只负责"键 → TID"，回表取实际元组、判可见性是 Table AM 的事。Index-Only Scan 则在 VM 标记 all-visible 的页上跳过回表。

---

## 5. 六种索引一览

| AM | 适用 | 结构要点 | 扫描模式 | 文档 |
|----|------|---------|---------|------|
| **B-tree** | 等值/范围/排序（最通用） | Lehman-Yao 高并发平衡树 | gettuple+bitmap，有序 | [13-nbtree](13-nbtree.md) |
| **Hash** | 仅等值 | 桶 + overflow 页，WAL 日志化 | gettuple | [14-other-indexes](14-other-indexes.md) |
| **GiST** | 几何/范围/KNN/全文 | 通用平衡搜索树，可定制谓词 | gettuple+bitmap，支持 ORDER BY 距离 | [14](14-other-indexes.md) |
| **SP-GiST** | 四叉树/基数树/不平衡分区 | 空间分区树 | gettuple+bitmap | [14](14-other-indexes.md) |
| **GIN** | 数组/全文/jsonb（多值） | 倒排索引：entry tree + posting tree | 仅 bitmap | [14](14-other-indexes.md) |
| **BRIN** | 大表自然有序列 | 块范围 min/max 摘要，极小 | 仅 bitmap | [14](14-other-indexes.md) |

---

## 6. 设计模式

- **对称抽象（与 Table AM 并列）**：表与索引各有一套 `*AmRoutine` 虚表 + `pg_am` 注册，规划器/执行器面向接口编程。
- **能力标志驱动规划**：用 `amcanorder`/`amsearcharray` 等布尔位声明式地告诉规划器索引支持什么，规划器据此决定能否用它满足 ORDER BY、下推 IN 等。
- **opclass 解耦类型与结构**：索引结构与数据类型语义正交，靠 `pg_amop/pg_amproc` 在运行期绑定，支撑任意类型与用户扩展。
- **两种扫描接口（gettuple/getbitmap）**：同一索引可服务"有序逐条"和"批量位图"两种执行策略，匹配不同选择率与查询形态。
- **职责单一（键→TID）**：索引只管定位，回表与可见性交给 Table AM，使索引实现无需懂 MVCC 细节。

---

## 7. 架构编排

```
规划器：amcostestimate 估代价 + 能力标志 → 选 IndexScan/IndexOnlyScan/BitmapHeapScan
执行器：
  IndexScan        → amgettuple  → TID → table_index_fetch_tuple → 可见性 → 输出
  BitmapIndexScan  → amgetbitmap → TID 位图 →（BitmapAnd/Or 组合）→ BitmapHeapScan 回表
建索引：ambuild（全表扫描排序灌入）
VACUUM：ambulkdelete + amvacuumcleanup（删除指向死 TID 的条目）
```

---

## 8. 动手探索

```sql
SELECT amname, amtype FROM pg_am WHERE amtype='i';   -- 列出索引 AM

-- 查某类型在 btree 下的 opclass、策略算子
SELECT opcname FROM pg_opclass o JOIN pg_am a ON a.oid=o.opcmethod
WHERE amname='btree' AND opcintype='int4'::regtype;

-- 观察 IndexScan vs BitmapHeapScan 的选择
EXPLAIN SELECT * FROM t WHERE id = 1;          -- Index Scan（高选择率）
EXPLAIN SELECT * FROM t WHERE id < 100000;     -- Bitmap Heap Scan（低选择率）
```

---

## 相关模块

- 表侧抽象：[10-table-am](10-table-am.md)
- B-tree 细节：[13-nbtree](13-nbtree.md)
- 其它索引：[14-other-indexes](14-other-indexes.md)
- 规划器选择：[../query/05-planner-paths-joins](../query/05-planner-paths-joins.md)
