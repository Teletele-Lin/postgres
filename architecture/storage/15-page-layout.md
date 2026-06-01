# Page Layout — 页面布局与元组格式

> 源码：`src/backend/storage/page/bufpage.c`、`src/include/storage/bufpage.h`
> 元组头：`src/include/access/htup_details.h`

PostgreSQL 所有持久化数据（表、索引、FSM、VM）都以**固定大小页（默认 8KB，`BLCKSZ`）** 为单位读写。理解页面布局是理解存储、MVCC、WAL 的共同基础。

---

## 1. 职责

定义磁盘块在内存中的标准布局：页头、行指针数组、空闲空间、元组数据、special 空间。提供 `PageInit`/`PageAddItem`/`PageGetItem` 等通用操作，被 heap 与所有索引 AM 共用。

---

## 2. 页面总体布局

`bufpage.h:26-47` 的示意（"slotted page"结构）：

```
 +----------------+---------------------------------+
 | PageHeaderData | linp1 linp2 linp3 ...           |  ← ItemId（行指针）数组，向后增长
 +-----------+----+---------------------------------+
 | ... linpN |                                      |
 +-----------+--------------------------------------+
 |           ^ pd_lower                             |
 |                                                  |
 |             空闲空间（free space）                |
 |                                                  |
 |             v pd_upper                           |
 +-------------+------------------------------------+
 |             | tupleN ... tuple3 tuple2 tuple1   |  ← 元组数据，从尾部向前增长
 +-------------+------------+-----------------------+
 |       ...   special space |                      |
 +---------------------------+----------------------+
                             ^ pd_special
```

**双向生长**：行指针从页头之后**向后**增长（`pd_lower` 推进），元组数据从 special 之前**向前**增长（`pd_upper` 回退）。两者相遇即页满（`bufpage.h:46-47`）。这种布局让行指针稳定（逻辑 offset 不变）、元组可在 `pd_upper..pd_special` 区域整理移动。

---

## 3. 核心数据结构

### 3.1 PageHeaderData

`src/include/storage/bufpage.h:184`：

```c
typedef struct PageHeaderData {
    PageXLogRecPtr pd_lsn;      // 最后修改本页的 WAL 记录 LSN ★WAL 先行的判据  bufpage.h:187
    uint16      pd_checksum;    // 页校验和（开启 data checksums 时）           bufpage.h:189
    uint16      pd_flags;       // 标志位（PD_HAS_FREE_LINES / PD_ALL_VISIBLE...）
    LocationIndex pd_lower;     // 空闲区起点 = 行指针数组末尾                  bufpage.h:146
    LocationIndex pd_upper;     // 空闲区终点 = 元组数据起点                    bufpage.h:147
    LocationIndex pd_special;   // special 空间起点                            bufpage.h:148
    uint16      pd_pagesize_version;  // 页大小 + 版本号
    TransactionId pd_prune_xid; // 本页最老的可 prune XID 提示（HOT）
    ItemIdData  pd_linp[FLEXIBLE_ARRAY_MEMBER];  // 行指针数组
} PageHeaderData;
```

**`pd_lsn` 是 WAL 先行规则的物理锚点**：buffer manager 刷脏页前，必须确保 WAL 已 flush 到 `>= pd_lsn`（见 [16-buffer-manager](16-buffer-manager.md)、[../transaction/33-wal](../transaction/33-wal.md)）。

### 3.2 ItemId（行指针 / line pointer）

`src/include/storage/itemid.h`，每个 4 字节：

```c
typedef struct ItemIdData {
    unsigned lp_off:15;    // 元组在页内的字节偏移
    unsigned lp_flags:2;   // 状态
    unsigned lp_len:15;    // 元组长度
} ItemIdData;
```

`lp_flags`：
- `LP_UNUSED`（0）：空闲，可复用。
- `LP_NORMAL`（1）：指向一个元组（`lp_off`/`lp_len` 有效）。
- `LP_REDIRECT`（2）：HOT 链重定向，`lp_off` 是另一个 ItemId 的下标。
- `LP_DEAD`（3）：元组已死，空间待回收。

**间接层的价值**：外部（索引、ctid）用 `(块号, 行指针下标)` 即 **TID** 引用元组；元组实际字节位置存在 ItemId 里。于是页内整理（prune/碎片整理）可移动元组、改 ItemId 的 `lp_off`，而 TID 不变——这是 HOT、单页 prune 的物理基础（见 [11-heap](11-heap.md)）。

### 3.3 HeapTupleHeaderData

`src/include/access/htup_details.h`，每个堆元组的头：

```c
struct HeapTupleHeaderData {
    union {
        HeapTupleFields t_heap;     // 含 t_xmin/t_xmax/t_cid
        DatumTupleFields t_datum;
    } t_choice;
    ItemPointerData t_ctid;         // (块,偏移)：指向自己；更新后指向新版本    ★更新链/HOT 链
    uint16   t_infomask2;           // 列数 + HEAP_HOT_UPDATED/HEAP_ONLY_TUPLE...
    uint16   t_infomask;            // HINT bits：HEAP_XMIN_COMMITTED/HEAP_XMAX_INVALID/HEAP_HASNULL...
    uint8    t_hoff;                // 用户数据起始偏移（头 + null bitmap 对齐）
    /* 之后是可选的 null bitmap，再之后是用户列数据 */
};
```

其中 `t_heap` 含 MVCC 关键字段：

```c
typedef struct HeapTupleFields {
    TransactionId t_xmin;   // 插入此版本的事务 XID
    TransactionId t_xmax;   // 删除/锁定此版本的事务 XID（0=未删）
    union { CommandId t_cid; TransactionId t_xvac; } t_field3;
} HeapTupleFields;
```

`t_xmin`/`t_xmax` + `t_infomask` 的 HINT bits 是 MVCC 可见性判断的全部依据（见 [../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)）。

---

## 4. 核心操作

### 4.1 PageInit

`PageInit()`（`bufpage.c:41`）初始化空页：清零，设 `pd_lower = SizeOfPageHeaderData`、`pd_upper = pd_special = BLCKSZ - specialSize`、写版本号。`specialSize` 由 AM 决定（heap=0，btree=`BTPageOpaqueData` 大小）。

### 4.2 PageAddItem

`PageAddItemExtended()`：在 `pd_upper` 前分配 `MAXALIGN(size)` 字节放元组，分配/复用一个 ItemId 指向它，推进 `pd_lower`、回退 `pd_upper`。可指定覆盖某个 `LP_UNUSED`/`LP_NORMAL` 槽。

### 4.3 校验和

开启 `data checksums` 时，`pd_checksum` 在**写盘前**由 `PageSetChecksumCopy` 计算（基于页内容 + 块号），读盘后校验，检测静默磁盘损坏。校验和不写进 WAL（每次刷盘重算）。

---

## 5. special 空间：留给 AM 的私有区

页尾 `pd_special..BLCKSZ` 由各 AM 自定义：
- heap：无（specialSize=0）。
- btree：`BTPageOpaqueData`（left/right link、level、flags，见 [13-nbtree](13-nbtree.md)）。
- gin/gist/spgist/hash：各自的页内元信息（兄弟指针、标志等）。

`PageGetSpecialPointer(page)` 取 special 区起点，AM 把它 cast 成自己的 opaque 结构。

---

## 6. TOAST —— 超长字段的页外存储

一个元组不能跨页，但页只有 8KB。当一行超过约 2KB（`TOAST_TUPLE_THRESHOLD`），超长的变长字段被 **TOAST**（The Oversized-Attribute Storage Technique）处理：压缩，或切块存到关联的 TOAST 表（每张大表自动有一个），主元组里只留指针。读取时按需"detoast"。这让逻辑上的大字段（text/bytea/jsonb）能存进固定页模型，对上层透明。

---

## 7. 设计模式

- **Slotted page（行指针间接层）**：逻辑 TID 与物理偏移解耦，使页内整理、HOT、prune 不影响外部引用——存储引擎最关键的物理设计。
- **双向生长 + 单一空闲区**：行指针与元组从两端相向增长，空闲管理简化为 `pd_lower/pd_upper` 两个游标。
- **页内自描述（pd_lsn/checksum/version）**：每页携带恢复（LSN）、完整性（checksum）、兼容性（version）信息，使页可独立校验与回放。
- **special 空间下放给 AM**：通用页框架 + AM 私有尾区，一套 `bufpage` 服务所有 heap/index AM。
- **TOAST 透明溢出**：固定页大小的限制用"超长字段外置 + 指针"化解，对 SQL 层不可见。

---

## 8. 架构编排

```
访问方法（heap/nbtree/...）
  │ PageInit / PageAddItem / PageGetItem / PageGetSpecialPointer
  ▼  bufpage.c
Page（== 内存中 8KB buffer 的内容指针）
  │ 由 Buffer Manager 提供（pin 住的 buffer）
  ▼  见 16-buffer-manager
BufferDesc.tag = (rel, fork, blocknum) ↔ 磁盘块
```

每次修改页面前 `MarkBufferDirty`，并在临界区内写 WAL、用 `PageSetLSN` 更新 `pd_lsn`。

---

## 9. 动手探索

```sql
CREATE EXTENSION pageinspect;

-- 页头
SELECT lower, upper, special, pagesize, lsn FROM page_header(get_raw_page('t', 0));

-- 行指针与元组头
SELECT lp, lp_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid,
       t_infomask::bit(16), t_infomask2::bit(16)
FROM heap_page_items(get_raw_page('t', 0));

-- TOAST
SELECT reltoastrelid::regclass FROM pg_class WHERE relname='t';
SELECT pg_column_size(big_text_col) FROM t;   -- 压缩/外置后的大小
```

---

## 相关模块

- 缓冲管理：[16-buffer-manager](16-buffer-manager.md)
- heap 元组语义：[11-heap](11-heap.md)
- 可见性：[../transaction/32-mvcc-snapshot](../transaction/32-mvcc-snapshot.md)
- WAL 与 LSN：[../transaction/33-wal](../transaction/33-wal.md)
- btree special：[13-nbtree](13-nbtree.md)
