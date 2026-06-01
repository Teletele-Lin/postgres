# 其它索引：GIN / GiST / SP-GiST / BRIN / Hash

> 源码：`src/backend/access/{gin,gist,spgist,brin,hash}/`
> 各自 README：`src/backend/access/<am>/README`
> 抽象层：[12-index-am](12-index-am.md)　对比：[13-nbtree](13-nbtree.md)

B-tree 之外的五种索引各为特定数据形态而生。本文给出每种的适用场景、结构要点、核心算法与设计取舍。

---

## 1. GIN — Generalized Inverted Index（倒排索引）

> 源码 `gin/`，文档 `gin/README`

### 适用
**一个值包含多个可检索键**的类型：数组（`@>`/`&&`）、全文检索 `tsvector`（`@@`）、`jsonb`（`?`/`@>`）、trigram（`pg_trgm`）。

### 结构（`README:17-26`）
倒排索引存储 `(key, posting list)` 对，posting list 是"包含该 key 的堆行集合"。两层结构：
- **entry tree**：对所有 key 建的 B-tree（key → 该 key 的 posting）。
- **posting list / posting tree**：某 key 的 TID 集合。少则内联成 **posting list**（`README:25-26`）；多则溢出为 **posting tree**（一棵以 TID 为序的 B-tree，`README:96`）。

### 核心算法
- **插入慢、查询快**：一行若含 N 个 key，要更新 N 个倒排条目，写放大明显。
- **pending list 缓冲**（`fastupdate`）：新条目先攒进无序的 pending list，延迟合并进主结构，批量摊薄写代价；查询时需同时扫 pending list。`gin_clean_pending_list()` / autovacuum 负责合并。
- 多 key 查询（如 `@>` 多元素）在各 key 的 posting 间做合并（AND/OR），只支持 `amgetbitmap`（吐 TID 位图）。

### 设计取舍
以**写放大 + pending list 缓冲**换取多值检索能力；倒排 + posting tree 让"一个键对应海量行"也能高效。

---

## 2. GiST — Generalized Search Tree（通用搜索树）

> 源码 `gist/`，文档 `gist/README`

### 适用
**可定义"包含/重叠"谓词**的类型：几何（`box`/`point` 的 `&&`/`<@`）、范围类型、`ltree`、全文、最近邻（KNN）。

### 结构
平衡树，每个内部节点存一个 **predicate（谓词键）**，是其子树所有项的"摘要/包络"（如几何里是包围盒 MBR）。查询时若谓词与查询条件不重叠则**剪枝整棵子树**。

### 核心算法（opclass 提供 7 个支持函数）
- `consistent`：判断查询是否可能匹配某节点谓词（决定是否下探）。
- `union`：合并若干键成一个覆盖谓词（建/分裂时算父键）。
- `penalty`：插入时选"代价最小"的子树（如包围盒膨胀最小）。
- `picksplit`：节点满时如何二分。
- `compress/decompress`、`same`。
- **KNN（`amcanorderbyop`）**：`ORDER BY geom <-> point`，用优先队列按距离下界遍历，可流式返回最近邻。

### 设计取舍
把"如何比较/包络"完全外包给 opclass 的 7 个函数 → 一棵树骨架适配任意空间/范围语义。代价是不保证全局有序、键设计影响性能。

---

## 3. SP-GiST — Space-Partitioned GiST（空间分区树）

> 源码 `spgist/`

### 适用
**非平衡、空间分区**结构天然适配的数据：四叉树（point）、k-d 树、基数树/前缀树（text、inet）。

### 结构
支持**不等高**的分区树。节点把空间**不重叠地**划分给子节点（与 GiST 的可重叠包络相反）。一个物理页可容纳多个树节点（inner tuple + leaf tuple）。

### 核心算法（opclass 5 函数）
`config`、`choose`（新值走哪个分区）、`picksplit`、`inner_consistent`/`leaf_consistent`（剪枝）。不重叠分区使得查询路径通常唯一，深度可变。

### 设计取舍
适合"分区互斥"的数据，比 GiST 在这类场景更紧凑高效；但要求划分不重叠，不是所有类型都适用。

---

## 4. BRIN — Block Range Index（块范围索引）

> 源码 `brin/`，文档 `brin/README`

### 适用
**大表 + 列值与物理存储顺序强相关**（如按时间追加的日志表的时间戳列）。

### 结构（`README:6-13`）
不索引每一行，而是对**连续 N 个堆页（一个 page range，默认 128 页）** 存一份摘要 tuple（如该范围内的 min/max）。摘要随插入更新。另有 **revmap**（range map）把"页范围 → 摘要 tuple"建索引。

### 核心算法
- 查询 `WHERE ts BETWEEN ...`：扫 revmap，对每个范围用摘要判断"是否可能含匹配行"，只对**可能匹配**的范围回表（`amgetbitmap` 吐这些范围的所有页）。
- 摘要 opclass：`minmax`（最常用）、`minmax_multi`（多区间，抗离群值）、`inclusion`（几何包络）、`bloom`（等值过滤）。
- **summarize**：新页范围由 `brin_summarize_new_values()` / autovacuum 补摘要。

### 设计取舍
**体积极小**（一张大表的 BRIN 可能只有几十 KB）、维护轻；代价是"假阳性"——摘要粗，匹配范围内仍需逐行过滤。仅当数据物理有序时收益大，否则每个范围 min/max 跨度过大形同虚设。

---

## 5. Hash — 哈希索引

> 源码 `hash/`

### 适用
**仅等值查询**（`=`），不支持范围、不支持排序（`amcanorder=false`）。

### 结构
哈希桶 + overflow 页 + bitmap 页（追踪 overflow 页使用）。桶数随数据增长**线性扩展**（split），用 highmask/lowmask 决定键落桶。

### 核心算法
- 对索引键算 hash，定位桶，桶内（含 overflow 链）线性找匹配。
- PostgreSQL 10 起 **完整 WAL 日志化**（之前 hash 索引不写 WAL、不可崩溃恢复、不能用于复制），现已是一等公民。

### 设计取舍
等值查询下哈希索引体积可比 B-tree 小、单点查询快；但功能单一（无范围/排序/唯一约束的某些特性），实践中 B-tree 常已够好，故使用较少。

---

## 6. 横向对比

| 维度 | B-tree | Hash | GiST | SP-GiST | GIN | BRIN |
|------|--------|------|------|---------|-----|------|
| 等值 | ✓ | ✓ | ✓(部分) | ✓ | ✓ | ✓(粗) |
| 范围/排序 | ✓✓ | ✗ | ✓ | ✓ | ✗ | ✓(粗) |
| 多值(数组/全文) | ✗ | ✗ | ✓ | △ | ✓✓ | ✗ |
| KNN | ✗ | ✗ | ✓ | ✓ | ✗ | ✗ |
| 索引体积 | 中 | 中 | 中 | 中 | 大 | 极小 |
| 写开销 | 中 | 中 | 中 | 中 | 大 | 极小 |
| 扫描模式 | gettuple+bitmap | gettuple | gettuple+bitmap | gettuple+bitmap | bitmap | bitmap |

---

## 7. 共通设计模式

- **opclass 外包语义**：GiST/SP-GiST/GIN/BRIN 都把"如何比较/包络/分区/摘要"交给 opclass 的支持函数，索引骨架与数据类型彻底解耦——这是 PostgreSQL"可扩展索引"的统一范式。
- **摘要/剪枝（GiST/SP-GiST/BRIN）**：内部节点存子树的覆盖摘要，查询靠"摘要与条件不交则整片剪掉"换取次线性扫描。
- **倒排 + 溢出升级（GIN）**：小集合内联、大集合升级为子树，自适应"一键多行"的分布。
- **延迟批量写（GIN pending list / BRIN summarize）**：把昂贵的结构维护推迟、批量化，摊薄写放大。
- **粗粒度换体积（BRIN）**：放弃逐行精度，用块级摘要把索引压到极小，靠回表过滤兜底精度。

---

## 8. 动手探索

```sql
-- GIN 全文/数组
CREATE INDEX ON docs USING gin (to_tsvector('english', body));
CREATE INDEX ON t USING gin (tags);          -- tags int[]，支持 @>
SET gin_pending_list_limit = '4MB';

-- GiST 几何 / 范围 / KNN
CREATE INDEX ON places USING gist (location);
EXPLAIN SELECT * FROM places ORDER BY location <-> point(0,0) LIMIT 5;  -- KNN

-- BRIN 大表有序列
CREATE INDEX ON events USING brin (created_at) WITH (pages_per_range=128);
SELECT * FROM brin_page_items(get_raw_page('events_created_at_idx', 2),
                              'events_created_at_idx');  -- pageinspect

-- Hash
CREATE INDEX ON t USING hash (email);
```

---

## 相关模块

- 抽象层：[12-index-am](12-index-am.md)
- B-tree：[13-nbtree](13-nbtree.md)
- 页面格式：[15-page-layout](15-page-layout.md)
