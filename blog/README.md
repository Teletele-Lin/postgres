# 深入 PostgreSQL 内核（博客系列）

> 一个数据库内核工程师的源码漫游笔记。每篇从一个**真实问题**出发，看 PostgreSQL 给出的解法，
> 配真实可复现的实验（`EXPLAIN` / `pageinspect` / `pg_waldump` / `gdb`）。
>
> 配套有一套更"词典式"的[架构参考文档](../architecture/README.md)——博客负责讲故事、给直觉，参考文档负责给精确的结构体、函数签名与 `文件:行号`。两者配合食用。

适用版本：PostgreSQL **19devel**（master 分支）。

---

## 阅读地图

系列分四条主线，可按兴趣跳读，但建议先读 0、1 两篇建立全局直觉。

**序章**
- [00 · 为什么要读 PostgreSQL 源码](00-prologue.md) — 多进程模型这第一个"反直觉"，与全局地图
- [01 · 一条 SQL 的奇幻漂流](01-journey-of-a-sql.md) — 用一条真实查询串起 parse→analyze→plan→execute 全流程 ⭐

**主线一：一条查询如何被执行**（查询处理）
- 02 · 解析器：为什么语法分析不许碰数据库
- 03 · 优化器（上）：它在搜索一个多大的空间
- 04 · 优化器（下）：代价模型与 join 顺序
- 05 · 火山模型：执行器如何"按需拉取"
- 06 · 表达式不是树：一次编译、多次执行

**主线二：数据如何落盘、如何并发**（存储与并发）
- 07 · 8KB 一页：堆表与元组的物理布局
- 08 · 缓冲池：时钟扫描与 WAL 先行
- 09 · B-tree：无锁查找是怎么做到的
- 10 · 四层锁：从自旋到死锁检测

**主线三：崩溃了怎么办、并发读写为何不打架**（事务）
- 11 · MVCC：写不阻塞读的秘密 ⭐
- 12 · WAL：先写日志，后写数据
- 13 · 检查点与崩溃恢复：幂等重放的艺术

**主线四：多进程如何协作**（进程与复制）
- 14 · Postmaster：一个进程崩了，为什么全体重启
- 15 · 并行查询：多进程没有共享堆，怎么并行
- 16 · 复制：把 WAL 流给另一台机器

> 标 ⭐ 的是已完成的样板长文，其余按系列推进逐步发布。

---

## 动手环境（强烈建议边读边跑）

本系列所有实验都基于一个固定的演示数据集。先把它建出来，后文不再重复。

```bash
# 1. 编译安装（Meson）
ninja -C build install

# 2. 初始化并启动一个全新集群
rm -rf $PGDATA && initdb -D $PGDATA
pg_ctl -D $PGDATA -l $PGDATA/start.log start

# 3. 进入 psql
psql postgres
```

```sql
-- 4. 安装窥探内核用的扩展
CREATE EXTENSION IF NOT EXISTS pageinspect;     -- 看页面/元组的物理字节
CREATE EXTENSION IF NOT EXISTS pg_buffercache;  -- 看缓冲池里有什么
CREATE EXTENSION IF NOT EXISTS pg_visibility;   -- 看可见性位图
CREATE EXTENSION IF NOT EXISTS pgstattuple;     -- 看死元组/膨胀

-- 5. 建演示数据集：customers/orders 做 join，big 做大表扫描/并行
CREATE TABLE customers (
    id    int PRIMARY KEY,
    name  text NOT NULL,
    city  text,
    vip   boolean DEFAULT false
);
CREATE TABLE orders (
    id          int PRIMARY KEY,
    customer_id int REFERENCES customers(id),
    amount      numeric(10,2),
    created_at  timestamptz DEFAULT now()
);
INSERT INTO customers
SELECT g, 'cust_'||g, (ARRAY['Beijing','Shanghai','Shenzhen','Hangzhou'])[1+g%4], g%10=0
FROM generate_series(1,1000) g;
INSERT INTO orders
SELECT g, 1+(g%1000), (random()*1000)::numeric(10,2), now()-(g||' minutes')::interval
FROM generate_series(1,50000) g;
CREATE INDEX orders_customer_id_idx ON orders(customer_id);
CREATE INDEX customers_city_idx ON customers(city);

CREATE TABLE big (id int, k int, v text);
INSERT INTO big SELECT g, g%1000, md5(g::text) FROM generate_series(1,2000000) g;

ANALYZE;
```

建完后的规模（后文实验输出都基于此）：

| 表 | 行数 | 页数 | 大小 |
|----|------|------|------|
| customers | 1,000 | 7 | 56 kB |
| orders | 50,000 | 319 | 2552 kB |
| big | 2,000,000 | 18,692 | 146 MB |

> 你机器上的具体数字（page 数、cost、计时）会因硬件、随机数据、PG 小版本而**略有不同**，但结构和量级一致。文中所有输出均为在上述环境真实运行所得，未经修饰。

---

## 关于"真实"

本系列拒绝"我觉得它大概是这样"。每个论断要么能在源码里指到（`文件:行号`），要么能在 psql 里跑出来。如果你跑出的结果和文中不符，欢迎对照——很可能是版本差异，也欢迎指出我的错误。
