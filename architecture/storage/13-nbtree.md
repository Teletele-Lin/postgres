# nbtree — B-tree 索引（Lehman-Yao 高并发）

> 源码：`src/backend/access/nbtree/`
> **必读** 设计文档：`src/backend/access/nbtree/README`
> 实现的接口：[12-index-am](12-index-am.md) 的 `IndexAmRoutine`

---

## 1. 职责

B-tree 是 PostgreSQL 最重要、最通用的索引：支持等值、范围、排序（`amcanorder`）、唯一约束、多列、IN 列表下推。它实现了对**有序可比较类型**的对数级查找，并在高并发下保证正确性。

实现基于 **Lehman & Yao（1981）** 的高并发 B-tree 算法（`README:8-12`），删除逻辑借鉴 Lanin & Shasha（1986）的简化版。

---

## 2. 目录结构

```
src/backend/access/nbtree/
  nbtree.c       IndexAmRoutine（bthandler）：btbuild/btinsert/btgettuple/...
  nbtsearch.c    _bt_search 下降查找、_bt_moveright 处理并发分裂
  nbtinsert.c    _bt_doinsert/_bt_insertonpg/_bt_split 插入与分裂
  nbtsplitloc.c  分裂点选择（_bt_findsplitloc）
  nbtpage.c      页面管理：_bt_getroot、meta 页、页面删除/回收
  nbtutils.c     scankey 构造、array key、suffix truncation
  nbtdedup.c     重复键去重（deduplication）
  nbtxlog.c      WAL 回放
  nbtsort.c      建索引：外部排序后自底向上批量装载
```

---

## 3. 核心数据结构

### 3.1 页面 special area：BTPageOpaqueData

每个 B-tree 页面尾部的 special space（见 [15-page-layout](15-page-layout.md)）放 `BTPageOpaqueData`（`src/include/access/nbtree.h:63`）：

```c
typedef struct BTPageOpaqueData {
    BlockNumber btpo_prev;    // 左兄弟（P_NONE = 最左）       nbtree.h:65
    BlockNumber btpo_next;    // 右兄弟（P_NONE = 最右）★L&Y 关键  nbtree.h:66
    uint32      btpo_level;   // 层级，叶子=0                  nbtree.h:67
    uint16      btpo_flags;   // BTP_LEAF/BTP_ROOT/BTP_META/BTP_DELETED... nbtree.h:68
    BTCycleId   btpo_cycleid; // vacuum 周期 ID，检测并发分裂
} BTPageOpaqueData;
```

`btpo_next`（**right-link**）是 Lehman-Yao 的灵魂。`btpo_prev`（left-link）是 PostgreSQL 为支持**反向扫描**额外加的（`README:75-80`）。

### 3.2 页面类型

- **meta 页**（block 0，`BTP_META`）：存根页位置、树高度等元信息。
- **root / internal 页**：放 **pivot tuple**（分隔键 + downlink），仅用于导航，不指向堆元组（`README:31-40`）。
- **leaf 页**（`BTP_LEAF`，level 0）：放指向堆元组的索引项；通过 left/right link 串成有序双向链表。
- 每页有一个 **high key**：该页允许键的上界（`README:18-19`）。

---

## 4. 核心算法

### 4.1 Lehman-Yao：无读锁的并发查找

经典 B-tree 并发查找要锁住路径上的页防止被并发修改。L&Y 的洞见（`README:14-29`）：给每页加 **right-link** 和 **high key**，就能**检测**并发分裂，从而几乎不持读锁地查找。

下降时（`_bt_search`，`nbtsearch.c:100`）跟着 downlink 到子页后，先比较子页的 **high key** 与查找键：

```
若 查找键 > high key:
    说明该页在我们到达前被并发分裂了，目标键已移到右边
    → 沿 btpo_next（right-link）右移到新页（_bt_moveright）
    → 可能要重复多次（页被分裂多次）
```

这就是 `README:24-29` 描述的核心：right-link 提供一条"追上并发分裂"的路径，使读者无需锁住整条下降路径。PostgreSQL 因共享 buffer 仍做**页级读锁**（`README:63-68`）以防读某页时它正被改，但不锁整条路径，并发度远高于锁耦合（lock coupling）。

### 4.2 键唯一性与 heap TID tiebreaker

L&Y 要求每层键唯一（子树 S 的键范围满足 `Ki < v <= Ki+1`，`Ki` 必须**严格小于** v，`README:42-49`）。但用户数据可能有重复键。PostgreSQL 用 **heap TID 作为隐含的最后一个排序列**：逻辑重复键按 TID 排序，从而在 B-tree 内部"键+TID"全局唯一，满足 L&Y 不变量。

### 4.3 pivot tuple 与 suffix truncation

internal 页的 pivot tuple 只为导航存在（`README:31-40`）。PostgreSQL 做 **suffix truncation**（`README:51-55`）：分裂时生成的分隔键只保留"足以区分左右页"的前缀列，后缀列截断为哨兵值"负无穷"。好处：分隔键更短 → internal 页扇出更大 → 树更矮；且分隔键不必是真实数据，可来自早已被 VACUUM 删掉的元组。

### 4.4 插入与页面分裂

`_bt_doinsert` → `_bt_search` 定位叶子页 → `_bt_insertonpg`（`nbtinsert.c:1119`）。若页放得下直接插；放不下则 `_bt_split`（`nbtinsert.c:1489`）：

1. `_bt_findsplitloc`（`nbtsplitloc.c`）选分裂点（不一定对半，会考虑"单调递增插入"等模式优化空间利用）。
2. **先建右页**，把上半部分元组搬过去，右页的 `btpo_next` 指向原页旧的右兄弟，原页 `btpo_next` 指向新右页 —— 这一步让任何并发查找都能经 right-link 到达正确页（即使父指针还没更新）。
3. 更新原右兄弟的 `btpo_prev`（维护反向链，`README:77-80`）。
4. **再原子地把新分隔键 + 右页 downlink 插入父页**（可能递归触发父页分裂）。

"先链右页、后插父指针"的顺序是 L&Y 正确性的关键：分裂的中间状态对读者安全，因为 right-link 始终提供可达路径。

### 4.5 唯一约束检查

唯一索引插入时 `_bt_check_unique`：定位插入点后沿 right-link 扫描所有等值键，检查是否有"对当前快照可见"的活元组冲突。可能需要等待并发事务（插入了同键但未提交）结束再判定。

### 4.6 去重（deduplication）

`nbtdedup.c`：当叶子页将满且有大量重复键时，把多个同键项合并成一个 **posting list tuple**（一个键 + 多个 TID），延缓页面分裂、缩小索引。这对低基数列上的索引节省可观空间。

### 4.7 建索引：自底向上排序装载

`nbtsort.c`：`btbuild` 不逐条插入，而是先对全部 (key, TID) 做**外部排序**（`tuplesort.c`），再自底向上一层层批量填满叶子页和上层 pivot 页，生成几乎全满、无碎片的紧凑树，远快于逐条 `_bt_insert`。支持并行建索引（`amcanbuildparallel`）。

---

## 5. 设计模式

- **Lehman-Yao：用结构换并发**：right-link + high key 把"检测并发分裂"编码进数据结构，使读者免锁路径——以少量冗余指针换取高并发。
- **分裂的崩溃/并发安全顺序**：先建并链右页（对读者立即可达），再补父指针，使任意中间状态都可被 right-link 兜底。
- **隐含 tiebreaker（heap TID）**：把"键唯一"不变量用一个隐藏排序列强行满足，让真实数据的重复键也能套用 L&Y 理论。
- **suffix truncation：分隔键即"够用就好"**：导航键只需区分边界，截断后缀换取更大扇出、更矮的树。
- **批量构建优于逐条**：建索引走"排序 + 自底向上装载"，避免逐条插入的分裂与随机写。
- **去重作为延迟分裂手段**：posting list 把"多 TID 同键"压缩，针对低基数场景优化空间与分裂频率。

---

## 6. 架构编排

```
查找/扫描：
  btgettuple → _bt_first → _bt_search（带 high key + right-link 容错下降） nbtsearch.c:100
             → 叶子页定位 → 沿 btpo_next 顺序扫描叶子链 → 返回 TID
插入：
  btinsert → _bt_doinsert → _bt_search → _bt_insertonpg                    nbtinsert.c:1119
           → 满则 _bt_split（建右页→链 right-link→插父 downlink）           nbtinsert.c:1489
建索引：
  btbuild → tuplesort 全量排序 → 自底向上装载                              nbtsort.c
VACUUM：
  btbulkdelete 扫叶子删死 TID；空页经两阶段安全回收（btpo_cycleid 防并发）
```

---

## 7. 动手探索

```sql
CREATE EXTENSION pageinspect;

-- meta 页与树高
SELECT * FROM bt_metap('t_pkey');

-- 看某页是叶子还是内部、left/right link、high key
SELECT * FROM bt_page_stats('t_pkey', 1);
SELECT itemoffset, ctid, data FROM bt_page_items('t_pkey', 1) LIMIT 5;

-- 去重效果
SELECT * FROM bt_page_items('t_low_card_idx', 1);  -- 看 posting list

-- 有序性带来的免排序
EXPLAIN SELECT * FROM t ORDER BY id LIMIT 10;       -- Index Scan，无 Sort 节点
```

---

## 相关模块

- 抽象层：[12-index-am](12-index-am.md)
- 其它索引：[14-other-indexes](14-other-indexes.md)
- 页面格式：[15-page-layout](15-page-layout.md)
- 谓词锁（SSI 在 btree 上加细粒度锁）：[../concurrency/23-predicate-ssi](../concurrency/23-predicate-ssi.md)
