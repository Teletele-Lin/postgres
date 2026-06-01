# Catalog — 系统表、BKI 与 Bootstrap

> 源码：`src/include/catalog/`（定义）、`src/backend/catalog/`（操作）、`src/backend/bootstrap/`
> 缓存层见 [41-caches](41-caches.md)

---

## 1. 职责

PostgreSQL **把数据库的元数据存在数据库自己的表里**——这些就是系统表（system catalog）。表结构、列、类型、函数、索引、约束、权限……全部是 `pg_catalog` schema 下的普通表的行。本模块负责：定义这些系统表的结构与初始数据、在 `initdb` 时把它们建出来（bootstrap）、提供增删改它们的 C 接口。

"用数据库管理数据库"是 PostgreSQL 可扩展性的根基：建一个类型/函数/索引 = 往系统表插几行；这也是为什么用户能在运行时无缝扩展类型、操作符、索引方法。

---

## 2. 关键系统表速览

| 表 | 内容 | 典型字段 |
|----|------|---------|
| `pg_class` | 所有"关系"（表/索引/视图/序列/物化视图/TOAST） | relname, relkind, relam, reltuples, relpages |
| `pg_attribute` | 每个关系的每一列 | attname, atttypid, attnum, attnotnull |
| `pg_type` | 数据类型 | typname, typlen, typinput/typoutput, typrelid |
| `pg_proc` | 函数/过程 | proname, proargtypes, prorettype, prosrc |
| `pg_operator` | 运算符 | oprname, oprleft/oprright, oprcode |
| `pg_index` | 索引（键列、表达式、谓词） | indrelid, indkey, indpred |
| `pg_am` | 访问方法（heap/btree/gin…） | amname, amtype, amhandler |
| `pg_namespace` | schema | nspname |
| `pg_class` 等的 OID | 各对象的全局唯一标识 | `oid` 系统列 |

一切互相用 **OID（Object Identifier，32 位）** 引用：`pg_attribute.attrelid` → `pg_class.oid`，`pg_class.reltype` → `pg_type.oid`，等等。

---

## 3. 系统表的定义方式：CATALOG 宏 + BKI

### 3.1 一张系统表的定义

以 `pg_class.h:34` 为例：

```c
CATALOG(pg_class,1259,RelationRelationId) BKI_BOOTSTRAP BKI_ROWTYPE_OID(83,...) BKI_SCHEMA_MACRO
{
    Oid       oid;                                                    // 系统列 oid
    NameData  relname;                                               // 表名
    Oid       relnamespace BKI_DEFAULT(pg_catalog) BKI_LOOKUP(pg_namespace);  // pg_class.h:43
    Oid       reltype      BKI_LOOKUP_OPT(pg_type);
    Oid       relowner     BKI_DEFAULT(POSTGRES) BKI_LOOKUP(pg_authid);
    Oid       relam        BKI_DEFAULT(heap) BKI_LOOKUP_OPT(pg_am);    // pg_class.h:55
    ...
} FormData_pg_class;

typedef FormData_pg_class *Form_pg_class;                            // pg_class.h:160
```

宏的含义：
- **`CATALOG(name, oid, OidSymbol)`**：声明这是系统表 `pg_class`，表的固定 OID 是 `1259`，C 里用符号 `RelationRelationId` 引用。宏展开成 `typedef struct FormData_pg_class { ... }`（`pg_class.h:26`）。`FormData_pg_class` 就是该表一行的 C 结构，`Form_pg_class` 是指向它的指针——读系统表元组时 cast 成它直接访问字段。
- **`BKI_BOOTSTRAP`**：此表在 bootstrap 极早期就要建（因为建别的表都依赖它）。
- **`BKI_DEFAULT(x)`**：`.dat` 数据文件里该列省略时的默认值。
- **`BKI_LOOKUP(pg_xxx)`**：该列是指向另一系统表的 OID，`.dat` 里可写**名字**，由生成工具查成 OID（`_OPT` 表示可为 0）。
- **`BKI_ROWTYPE_OID`**：该表隐含的复合行类型的 OID。

### 3.2 初始数据：.dat 文件

每个 `pg_xxx.h` 旁边有 `pg_xxx.dat`，是 Perl 风格的初始行数据（如 `pg_proc.dat` 列出所有内置函数）。`initdb` 把这些行灌进刚建好的系统表。`genbki.pl`（`genbki.h`）在**构建期**读 `.h` 的 CATALOG 定义 + `.dat`，生成：
- `postgres.bki`：bootstrap 脚本（建表 + 灌初始数据的指令）。
- `schemapg.h`、各种 OID 宏：编译期常量（如 `RelationRelationId = 1259`）。

### 3.3 Bootstrap

`initdb` 启动一个特殊的 **bootstrap 模式** backend（`src/backend/bootstrap/bootstrap.c`），按 `postgres.bki` 脚本：先用硬编码方式创建 `BKI_BOOTSTRAP` 表（pg_class/pg_attribute/pg_proc/pg_type），再用它们注册其余系统表与初始对象。这是"先有鸡还是先有蛋"的解法——用最小硬编码引导出能自我描述的系统表集合，之后一切（包括建普通表）都走正常路径。

---

## 4. 操作系统表

`src/backend/catalog/`：
- `heap.c`：`heap_create_with_catalog()` —— 建一张新表时，往 `pg_class`/`pg_attribute`/`pg_type` 插入相应行、分配 OID、建文件。
- `index.c`：`index_create()` —— 建索引，写 `pg_class`/`pg_index`/`pg_attribute`。
- `pg_*.c`（如 `pg_proc.c`、`pg_type.c`、`pg_constraint.c`）：各系统表的高层插入/更新封装。
- `namespace.c`：schema 解析与 `search_path` 处理（把无限定名解析到具体对象）。
- `dependency.c`：对象依赖追踪（`pg_depend`），保证 DROP 时级联/拒绝正确。
- `catalog.c`：`IsSystemRelation` 等判定。

DDL 命令（`src/backend/commands/`）最终都落到这些函数上修改系统表。系统表本身也是 heap 表，用普通的 heap_insert/update + 索引维护，受 MVCC 与 WAL 保护——所以 DDL 也是事务性的、可回滚的（PostgreSQL 的一大优势）。

---

## 5. 设计模式

- **元数据即数据（自举的目录）**：用普通表存元数据，用同一套存储/MVCC/WAL/事务机制管理它们，使 DDL 事务化、可回滚，并让类型/函数/索引可在运行时扩展。
- **声明式定义 + 代码生成（CATALOG/BKI/genbki）**：系统表的 C 结构、OID 常量、bootstrap 脚本、初始数据全由 `.h` + `.dat` 单一来源生成，避免手工同步多处。
- **OID 统一寻址**：所有目录对象用 32 位 OID 互相引用，`BKI_LOOKUP` 让人写名字、机器存 OID。
- **Bootstrap 自举**：用最小硬编码建出 `BKI_BOOTSTRAP` 核心表，再由它们注册其余一切，解决元数据的"先有鸡蛋"问题。
- **Form 结构直接映射元组**：`Form_pg_class` 等让 C 代码把系统表元组当结构体读，零拷贝访问定长前缀字段。

---

## 6. 架构编排

```
构建期：
  pg_*.h（CATALOG 宏）+ pg_*.dat  ──genbki.pl──> postgres.bki + OID 宏 + schemapg.h

initdb：
  bootstrap backend 执行 postgres.bki
    → 硬编码建 BKI_BOOTSTRAP 表（pg_class/pg_attribute/pg_proc/pg_type）
    → 用它们注册其余系统表 + 灌 .dat 初始数据

运行期 DDL：
  CREATE TABLE → commands/tablecmds.c → catalog/heap.c heap_create_with_catalog
    → heap_insert 到 pg_class/pg_attribute/pg_type（普通 MVCC + WAL）
    → 发缓存失效消息（见 41-caches）
读取：通过 syscache/relcache 缓存（见 41-caches），而非每次扫系统表
```

---

## 7. 动手探索

```sql
-- 系统表就是普通表，可直接查
SELECT relname, relkind, relpages, reltuples FROM pg_class WHERE relname='t';
SELECT attname, atttypid::regtype, attnum FROM pg_attribute
WHERE attrelid='t'::regclass AND attnum>0;

-- OID 互相引用
SELECT c.relname, t.typname FROM pg_class c JOIN pg_type t ON c.reltype=t.oid
WHERE c.relname='t';

-- DDL 是事务性的（可回滚！）
BEGIN; CREATE TABLE tmp(a int); ROLLBACK;   -- tmp 不存在

-- 看某固定 OID
SELECT 'pg_class'::regclass::oid;   -- 1259
```

---

## 相关模块

- 缓存层：[41-caches](41-caches.md)
- 名称解析消费者：[../query/02-analyzer](../query/02-analyzer.md)
- 统计信息（pg_statistic）供优化器：[../query/05-planner-paths-joins](../query/05-planner-paths-joins.md)
- AM 注册（pg_am）：[../storage/10-table-am](../storage/10-table-am.md)
