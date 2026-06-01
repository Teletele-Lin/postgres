# Postmaster — 主守护进程与进程生命周期

> 源码：`src/backend/postmaster/postmaster.c`、`src/backend/tcop/postgres.c`
> 配套：[51-background-procs](51-background-procs.md)、[52-shmem-ipc](52-shmem-ipc.md)

---

## 1. 职责

Postmaster 是 PostgreSQL 服务器的**根进程**，由 `pg_ctl start` 启动。它负责：
- 监听客户端连接，为每个连接 **fork 一个 backend** 子进程。
- 启动并监管所有**辅助进程**（checkpointer、bgwriter、walwriter、autovacuum launcher、archiver、startup、walsender/receiver、逻辑复制 launcher 等）。
- 创建共享内存与信号量池。
- **在子进程崩溃时重置系统**，恢复可用状态。

它是协调者，不是工作者——真正干活的是它 fork 出来的子进程。

---

## 2. 核心设计：postmaster 不碰共享内存

`postmaster.c:15-23` 顶部注释道出关键设计：

> postmaster 在启动时创建共享内存与信号量池，但**作为规则它自己不碰它们**。特别地，它不是 backend 的 PGPROC 数组成员，因而不参与锁管理。让 postmaster 远离共享内存操作使它更简单、更可靠。它几乎总能通过**重置共享内存**从单个 backend 的崩溃中恢复；若它对共享内存做很多事，就会跟着 backend 一起崩。

这是**故障隔离**的精髓：
- 共享内存可能被某个崩溃的 backend 写坏（野指针）。postmaster 不读写共享内存，就不会被坏数据带崩。
- 当某 backend 异常退出（段错误等），共享内存可能处于不一致状态（持着锁、半改的 buffer）。postmaster 检测到后，**杀掉所有其他 backend、重置共享内存、重新初始化**，让系统回到干净状态——而不是冒险继续用可能损坏的共享状态。
- 代价：一个 backend 崩溃会导致**所有连接被中断**（但已提交数据安全，靠 WAL）。这是多进程隔离的合理权衡。

---

## 3. 连接处理：fork-per-connection

`postmaster.c:25-32`：收到连接请求后**立即 fork**，子进程先做认证、成功则变成 backend。好处：
- 认证代码可写成简单的单线程风格。
- 更重要：SSL、PAM 等**非多线程安全库**的阻塞不会拖累其他客户端（若在单进程里串行认证，一个慢认证会 DoS 所有人）。

流程：
```
监听 socket（listen）
  └─ 新连接 → fork() 子进程
        ├─ 父（postmaster）：记录子 pid，继续监听
        └─ 子：BackendInitialize 认证 → 成功 → PostgresMain 主循环
```

> 注：Windows 无 fork，用 `EXEC_BACKEND` 模式（重新 exec 自己 + 共享内存重附加）模拟。Unix 下 fork 共享父进程已附加的共享内存。

---

## 4. Backend 主循环：PostgresMain

`PostgresMain()`（`src/backend/tcop/postgres.c:4274`）是每个 backend 的核心，认证后进入无限循环读前端协议消息（见 [../00-overview](../00-overview.md) §3.3）：

```c
for (;;) {
    firstchar = ReadCommand(&input_message);   // 读一条 FE/BE 协议消息
    switch (firstchar) {
        case 'Q':  exec_simple_query(...);      // 简单查询协议
        case 'P':  exec_parse_message(...);     // 扩展协议：Parse
        case 'B':  exec_bind_message(...);      // Bind
        case 'E':  exec_execute_message(...);   // Execute
        case 'X':  /* Terminate，退出 */
        ...
    }
}
```

每条命令外包着 `StartTransactionCommand`/`CommitTransactionCommand`（见 [../transaction/30-xact](../transaction/30-xact.md)），错误经 `sigsetjmp` 在循环顶部捕获并 abort 当前事务后继续（见 [../infra/71-error-elog](../infra/71-error-elog.md)）。

---

## 5. 进程状态机与监管

postmaster 维护一个**状态机**（`PM_STARTUP` → `PM_RUN` → `PM_SHUTDOWN` 等）协调启动/关闭/恢复：
- **启动**：建共享内存 → 启 startup 进程做崩溃恢复（若需要，见 [../transaction/34-recovery-checkpoint](../transaction/34-recovery-checkpoint.md））→ 恢复完成后启各辅助进程、进入 `PM_RUN` 接受连接。
- **监管**：通过 `SIGCHLD` 感知子进程退出。辅助进程异常退出 → 视严重性重启或全体重置；backend 异常退出 → 触发崩溃恢复式重置。
- **关闭**：三种模式 —— smart（等连接结束）、fast（中断连接、回滚、干净关闭）、immediate（直接杀、下次启动走崩溃恢复）。

`pg_ctl`/`postmaster` 通过信号与 postmaster 通信触发这些转换。

---

## 6. 设计模式

- **协调者-工作者分离**：postmaster 只 fork 与监管，不做实际数据工作，把"管理"与"执行"彻底解耦。
- **故障隔离（远离共享内存）**：让根进程不接触可能被污染的共享内存，使其能在子进程崩溃后充当"重置者"而非陪葬者——多进程架构可靠性的支点。
- **fork-per-connection**：每连接独立进程，天然隔离崩溃、简化认证、规避非线程安全库的 DoS。
- **崩溃即重置**：任一 backend 异常 → 全体重置而非局部修补，宁可中断连接也不在可疑共享状态上继续——正确性优先。
- **进程状态机**：用显式状态机统一表达启动/恢复/运行/关闭，把多子进程的生命周期编排集中管理。

---

## 7. 架构编排

```
pg_ctl start
  └─ postmaster                                     postmaster.c
       ├─ 建共享内存 + 信号量（CreateSharedMemoryAndSemaphores，见 52）
       ├─ 启 startup 进程 → 崩溃恢复（如需）            见 transaction/34
       ├─ 进入 PM_RUN：
       │     ├─ 启辅助进程：checkpointer/bgwriter/walwriter/
       │     │              autovacuum launcher/archiver/...  见 51
       │     └─ 监听 socket
       │           └─ 新连接 → fork → 认证 → PostgresMain      tcop/postgres.c:4274
       │                                   └─ 命令循环（Q/P/B/E）→ 查询流水线
       └─ SIGCHLD 监管：
             ├─ 辅助进程退出 → 重启/重置
             └─ backend 崩溃 → 杀全体 + 重置共享内存 + 重新初始化
```

---

## 8. 动手探索

```bash
# 进程树：postmaster + 各辅助进程 + 每连接一个 backend
ps -ef | grep postgres
# postgres: checkpointer / background writer / walwriter /
#           autovacuum launcher / logical replication launcher /
#           <user> <db> <host>(port) idle    ← 这些是 backend

pg_ctl -D $PGDATA stop -m fast      # 三种关闭模式：smart/fast/immediate
```

```sql
SELECT pid, backend_type, state FROM pg_stat_activity;  -- backend_type 标明角色
SELECT pg_backend_pid();                                 -- 当前连接的 backend pid
```

---

## 相关模块

- 辅助进程详解：[51-background-procs](51-background-procs.md)
- 共享内存：[52-shmem-ipc](52-shmem-ipc.md)
- 命令循环下游：[../00-overview](../00-overview.md) §3、[../query/01-parser](../query/01-parser.md)
- 错误恢复：[../infra/71-error-elog](../infra/71-error-elog.md)
