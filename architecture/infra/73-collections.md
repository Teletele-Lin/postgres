# 基础容器 — List / Bitmapset / HTAB

> 源码：`src/backend/nodes/{list,bitmapset}.c`、`src/backend/utils/hash/dynahash.c`
> 头文件：`src/include/nodes/{pg_list,bitmapset}.h`、`src/include/utils/hsearch.h`

内核里最常用的三种通用容器。读任何子系统都会反复遇到它们，理解其语义与代价能极大提升读码效率。

---

## 1. List —— 最常用的列表

`src/include/nodes/pg_list.h`。承载几乎所有"一串东西"：RTE 列表、target list、Path 列表、参数列表……

### 1.1 结构（数组实现）

`pg_list.h:53`：

```c
typedef struct List {
    NodeTag    type;        // T_List / T_IntList / T_OidList / T_XidList   pg_list.h:55
    int        length;
    int        max_length;
    ListCell  *elements;    // ★可重分配的 cell 数组（不是链表！）          pg_list.h:58
    ListCell   initial_elements[FLEXIBLE_ARRAY_MEMBER];
} List;

typedef union ListCell {    // pg_list.h:45
    void *ptr_value;        // List（指针元素，最常见）
    int   int_value;        // IntList
    Oid   oid_value;        // OidList
    ...
} ListCell;
```

**重要历史**：List 曾是真正的链表，PostgreSQL 13 起改为**动态数组**（cell 连续存储）。所以：
- `list_nth(list, i)` 现在是 O(1)（过去 O(n)）。
- `lappend`（尾插）摊销 O(1)；`lcons`（头插）O(n)（要移动）。
- 遍历 cache 友好。

四种类型：`List`（元素是 Node 指针）、`IntList`、`OidList`、`XidList`（元素是标量）。`NIL` 是空列表。

### 1.2 常用 API

```c
foreach(lc, list) { Node *n = lfirst(lc); ... }   // 遍历
foreach_int(i, intlist) { ... }                    // 标量遍历（新式宏）
list = lappend(list, ptr);    list = lcons(ptr, list);
ptr = linitial(list);  ptr = llast(list);  ptr = list_nth(list, i);
list = list_concat(a, b);  list = list_delete_ptr(list, ptr);
len = list_length(list);
```

> 陷阱：`lappend` 可能 realloc，返回新指针——必须 `list = lappend(list, x)` 而非丢弃返回值。`foreach` 中删元素要小心数组移动。

---

## 2. Bitmapset —— 整数集合

`src/include/nodes/bitmapset.h`。用位图高效表示**小整数的集合**。

### 2.1 用途

到处表示"一组 ID"：
- `Relids`（`RelOptInfo.relids`）：一组 RT 下标（优化器靠它给 join rel 去重，见 [../query/05-planner-paths-joins](../query/05-planner-paths-joins.md)）。
- 一组列号（`rd_indexattr`、`chgParam`）。
- 各种"是否包含某成员"的快速判定。

### 2.2 API

```c
Bitmapset *bms_make_singleton(int x);
bms = bms_add_member(bms, x);   bms = bms_del_member(bms, x);
bool in = bms_is_member(x, bms);
bms = bms_union(a, b);  bms = bms_intersect(a, b);  bms = bms_difference(a, b);
bool sub = bms_is_subset(a, b);  bool eq = bms_equal(a, b);
int x = -1; while ((x = bms_next_member(bms, x)) >= 0) { ... }   // 遍历
```

内部是 `bitmapword[]`，集合操作 = 位运算，对"成员是密集小整数"的场景极省空间且快。`NULL` 表示空集（约定：空 Bitmapset 规范化为 NULL）。

---

## 3. HTAB —— 动态哈希表

`src/include/utils/hsearch.h`，实现 `dynahash.c`。通用哈希表，既能在进程本地内存也能在**共享内存**里建（buffer 哈希表、锁表都是 HTAB）。

### 3.1 创建与使用

```c
HASHCTL ctl = { .keysize = sizeof(K), .entrysize = sizeof(E), .hcxt = ctx };
HTAB *htab = hash_create("name", nelem, &ctl, HASH_ELEM | HASH_BLOBS | HASH_CONTEXT);

E *e = hash_search(htab, &key, action, &found);
//  action: HASH_FIND（查）/ HASH_ENTER（无则插）/ HASH_REMOVE（删）/ HASH_ENTER_NULL
// 遍历：
HASH_SEQ_STATUS seq; hash_seq_init(&seq, htab);
while ((e = hash_seq_search(&seq)) != NULL) { ... }
```

### 3.2 特点

- **可放共享内存**：`HASH_SHARED_MEM` + 预分配，配合分区锁（如 buffer 哈希表 128 分区，见 [../storage/16-buffer-manager](../storage/16-buffer-manager.md)）。
- **自定义 hash/match 函数**（`HASH_FUNCTION`/`HASH_COMPARE`）或用内置（`HASH_BLOBS` 二进制 key、`HASH_STRINGS` 字符串 key）。
- entry 内含 key（key 是 entry 结构的前缀），`hash_search` 返回整个 entry。
- 共享内存 HTAB **不能动态扩容**（启动时按上限预分配）；本地 HTAB 可增长。

此外还有更轻量的 `simplehash`（`src/include/lib/simplehash.h`，开放寻址、模板化，执行器哈希聚合/哈希 join 用它，性能更高但不可共享内存）。

---

## 4. 设计模式

- **List 数组化（cache 友好）**：把曾经的链表换成可重分配的连续数组，O(1) 随机访问 + 顺序遍历友好，以"尾插快、头插慢"的取舍服务最常见的追加模式。
- **类型化集合（List 的四变体 / Bitmapset）**：用专门的 Int/Oid/Xid List 与位图集合避免装箱与指针开销，针对标量与小整数集合优化。
- **位图表示稀疏语义（Bitmapset = Relids）**：把"一组关系/列"编码为位图，使集合运算 = 位运算，并让 join 子问题用 relids 做 key 去重成为可能。
- **统一哈希抽象（HTAB）跨内存域**：同一接口服务本地与共享内存哈希表，配分区锁支撑高并发——buffer/锁表复用同一基础设施。
- **按场景分层（HTAB vs simplehash）**：需要共享内存/通用性用 HTAB，追求单进程极致性能用 simplehash，按需取舍。

---

## 5. 在内核中的足迹

```
List      : rtable、targetList、pathlist、cteList、各种参数/子节点列表（无处不在）
Bitmapset : RelOptInfo.relids（优化器 join 去重）、chgParam（执行器 rescan）、列集合
HTAB      : 共享内存——buffer 哈希表、锁表(LOCK/PROCLOCK)、ShmemIndex、catcache 桶
simplehash: 执行器 HashAgg/HashJoin 的内存哈希表
```

---

## 6. 动手探索

```c
// 阅读建议：在 gdb 里
//   p *(List*)some_list            看 length/elements
//   call pprint(some_list)         若元素是 Node，打印整列
//   p bms_num_members(relids)      Bitmapset 成员数
```

```sql
-- List/Bitmapset 多为内部结构，间接观察：
EXPLAIN (VERBOSE) SELECT * FROM a,b,c WHERE a.x=b.x AND b.x=c.x;
-- 计划里 Output 列表、各节点的关系集合即这些容器的外在表现
```

---

## 相关模块

- 节点系统（List 是 Node）：[72-node-system](72-node-system.md)
- Relids 的使用：[../query/05-planner-paths-joins](../query/05-planner-paths-joins.md)
- 共享 HTAB 实例：[../storage/16-buffer-manager](../storage/16-buffer-manager.md)、[../concurrency/22-heavyweight-lock](../concurrency/22-heavyweight-lock.md)
