# ADR 0002：M0-02A PostgreSQL 并发技术验证

日期：2026-10-30。依据：分册一 §0.5 第 1 项、§2–4；大纲 §4.7、§4.3；总册 §7.4.1、§8.4。属于 HLD 推荐范围内的内部原型与验证，不更改跨组接口或冻结语义，没有新增依赖、业务表和迁移。

## 交付与结论

| 验证项 | 证据 | 结论 |
| --- | --- | --- |
| 四种行锁冲突 | 真实 PG16、两条独立 backend、16 种有序模式组合，冲突用 NOWAIT 验证 55P03 | KEY SHARE 与 NO KEY UPDATE 兼容；FOR UPDATE 阻塞其他四种模式 |
| 普通 UPDATE 的语义 | 在非键列 UPDATE 未提交时实际请求 KEY SHARE 与 UPDATE | 普通非键更新不替代永久撤回所需的显式 FOR UPDATE |
| SKIP LOCKED | 锁住第 1 行时候选为第 2 行，释放后候选为第 1 行 | 仅证明跳过锁定候选；同键 FIFO 仍需任务表和键占用协议 |
| 事务局部配置 | 分别提交和回滚 SET LOCAL lock_timeout | 两种结局均恢复原会话值 |
| Read Committed 重读 | 读者先读 0；通过 PG 锁依赖确认等待；写者提交后重新汇总为 10 | 拿锁后的独立语句可读取新提交；不能复用取锁前汇总 |
| 真实死锁与整事务重试 | 两个命名暂停点控制反向取锁，观察实际等待后形成环；捕获 40P01 | 仅一个事务成为受害者；事务外捕获后已回滚，完整重试后每个效果仅保留一次 |
| 统一顺序 | 两个事务按相同顺序，经守卫登记后显式 SQL 加锁；PG 确认真实竞争 | 只有等待，没有死锁和重复写入 |

## 锁序守卫原型

实现位于 `runtime/core/lock_order.py`，没有数据库依赖或 SQL 执行。每个事务使用一个实例，按 HLD 顺序记录拟加锁对象；事务失败或结束后丢弃，不跨事务复用。

顺序为任务 → 键占用 → 文档/账号控制 → 来源控制（仅相应控制分支）→ 交接快照 → 撤回计数 → blob → 逻辑对象 → 一组状态 → I9 观测/核验 → I21 定时 → I10 派生写入 → 输出序号 → 作业/检查点。

- 同层资源采用调用方给出的非空字符串元组作字典序排序键；需要表/范围区分时将其纳入元组。同层新资源不能倒序。
- 同资源必须预先选择最强模式，禁止升级。重复声明相同或较弱模式不改变原有强锁声明，也不新增锁。
- `assert_held` 检查登记的资源与最低模式，不在异常里输出身份键。
- 该原型验证内部算法，不冻结后续 I5/I7/I9/I10/I21 的公开提交上下文。身份编码、根来源集合和调用挂点仍随 M0 两批契约落地。
- 必须在实际 SQL 之前调用守卫，SQL 失败后回滚并丢弃守卫。它不扫描任意 SQL，也不证明数据库已经持锁，不能用它替代写屏障或权限校验。

## 确定性测试工具

`tests/support/failpoints.py:PausePoint` 用 reached/released 两个 Event 完成一次性暂停；控制方确认到达后显式放行。所有等待有超时，测试用 finally 放行。

`tests/support/postgres.py:wait_for_blocker` 以 `pg_blocking_pids()` 返回的真实依赖为放行条件。循环中的 10ms 间隔仅降低轮询压力，事件先后不由这个时间决定。未观察到依赖则失败，不用固定 sleep 假定对方已经持锁。

每个 PG 实验使用随机 `m0_lock_probe_<uuid>` schema：行和效果表只在该命名空间内建立；关闭所有实验连接后只删除自己创建的 schema。无业务表、无 Alembic 迁移、无数据重置。连接级超时只用于短生命周期实验连接；生产连接池仍须按 HLD 使用事务局部设置。

## 执行与阶段边界

```bash
rtk proxy make check
rtk proxy env TEST_DATABASE_URL='postgresql://april:april_local_only@127.0.0.1:15432/april' make test-pg-concurrency
rtk proxy env TEST_DATABASE_URL='postgresql://april:april_local_only@127.0.0.1:15432/april' make test-integration
```

本地验证环境为 Python 3.12.12、psycopg 3.3.6、PostgreSQL 16.12。原 CI 的 `make test-integration` 自动包含新增实验；无须新增工作流或独立第三方测试框架。

本次完成 M0-02 的 PG 子项，不代表全部技术验证或 M0 出口。Kafka/JCS/CEL/契约工具链/进程隔离/真实来源协议仍有后续独立交付；死锁实验中的两次尝试不是生产重试预算实现。

参考：[PostgreSQL 16 锁与死锁](https://www.postgresql.org/docs/16/explicit-locking.html)、[pg_blocking_pids](https://www.postgresql.org/docs/16/functions-info.html)、[psycopg 事务上下文](https://www.psycopg.org/psycopg3/docs/basic/transactions.html)。实际结论以本项目实验与 PR CI 为准。
