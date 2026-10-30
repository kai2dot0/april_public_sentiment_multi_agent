# M0-02A：PostgreSQL 并发地基与技术验证

日期：2026-10-30（Asia/Shanghai）。工具：Codex(GPT)。分支：`feat/m0-postgres-concurrency-validation`。起点：已合并的 M0-01，main=`b6f64a98a3819db0ccf5e3b1b0f64a3cd00085f6`。

## 1. 需求与切分

用户要求继续下一步开发，每项功能/模块/设计独立 PR 并 merge，保留详细 dev_memory。本轮从 roadmap 的 M0-02 技术验证进入，先完整交付 PG 锁/守卫/命名屏障这一项，其他验证分项分别保留，避免将 M0-02 或 M0 出口误标完成。

依据：分册一 §0.5.1、§2、§3、§4；大纲 §4.7 的全局锁序及死锁规则、§4.3 的回收后重读；总册 §7.4.1、§8.4。推荐范围内的原型与验证，不增加第三方库，不修改公开契约、身份格式或数据库迁移，也不创建业务表。

本阶段继续采用 ponytail 最小完整实现原则。项目无 `.codegraph/`，先读规则和已有 Make/CI/测试，再使用 rg 定位；未启动子 Agent。

## 2. 开发方案

### 内部锁序守卫

`runtime/core/lock_order.py`：

- `LockStage` 按任务、键占用、文档/账号控制、来源控制、交接快照、撤回计数、blob、逻辑对象、一组状态、I9、I21、I10、输出序号、作业/检查点排序。
- 来源控制/交接快照仅供相应控制路径使用；普通内容引用不会因此自动锁来源行。
- `RowLockMode` 为 KEY SHARE / SHARE / NO KEY UPDATE / UPDATE；SQL 片段从封闭枚举产生，不收任意 SQL。
- `LockOrderGuard.before_lock(stage, key, mode)` 在实际 SQL 之前检查并登记拟加锁操作，拒绝逆序和任何升级。资源键为非空字符串元组，不自动将整数或不同编码转成字符串。
- 已持有资源可重复声明相同/较弱模式，保留原强锁，不推进新锁前沿；新资源按（阶段，排序键）单调。
- `assert_held` 校验登记资源的最低模式，异常不回显可能有个人标识的资源键。
- 每个事务新建实例；SQL 失败或事务结束后丢弃。守卫不是实际锁执行器，不证明 SQL 已成功，不扫描任意 SQL，不代替权限/撤回屏障。
- 此为 runtime 内部原型，不提前冻结后续跨组提交上下文；I5/I7/I9/I10/I21 仍按两批契约落地。将来接入实际提交时必须逐路径加调用点和断言测试。

### 命名屏障

`tests/support/failpoints.py:PausePoint`：工作线程 hit，控制方 wait_until_reached 后 resume。使用两个 Event、一次性入口锁与默认 10 秒超时；重复命中明确失败；finally 中幂等 resume，避免异常导致遗留线程。

`tests/support/postgres.py:wait_for_blocker`：通过只读 `pg_blocking_pids` 判断预期阻塞是否已经在 PG 中成立，最多等待 5 秒。10ms 轮询仅限流，不依赖固定延时建立顺序。命名暂停点先控制线程进度，再用实际锁依赖确认，之后才放行另一方。

### 真实数据库夹具

- 根 `tests/conftest.py` 提供 `postgres_dsn`，只读显式 `TEST_DATABASE_URL`，未提供时失败而非跳过，不回退业务 DATABASE_URL。
- 原 `tests/integration/test_postgres.py` 复用该配置；其原迁移/向量检查语义不变。
- `tests/integration/conftest.py` 每个实验创建随机 `m0_lock_probe_<uuid>` schema，包含 `probe_rows` 和 `probe_effects`，没有 Alembic 版本变化。
- `PostgresProbe` 每次 connect 都建立不同 backend，autocommit 外壳配显式 transaction()，设置 Read Committed。会话级 15 秒 statement / 8 秒 lock / 20 秒 idle-in-transaction 超时仅用于短生命周期实验连接，不用于正式连接池。
- finally 关闭全部实验会话后只 DROP 自己生成的 schema CASCADE。不能扫描/删除其他实验或业务命名空间。dsn 不进入对象 repr 或输出。

## 3. 实验与可复现断言

| 实验 | 构造 | 断言 |
| --- | --- | --- |
| 锁矩阵 16 对 | 两条独立连接，持锁方先加锁，竞争方 NOWAIT | 冲突仅按独立 Table 13.3 期望产生 55P03；兼容模式成功 |
| 非键 UPDATE | 未提交更新 value 后另一事务取锁 | KEY SHARE 成功，UPDATE 被拒；证明普通 UPDATE 不能替代永久撤回锁 |
| SKIP LOCKED | 锁住 id=1 后扫描最小候选 | 返回 2；释放后返回 1；不宣称同键 FIFO 已完成 |
| SET LOCAL 两条路径 | 事务内改 lock_timeout，分别提交/回滚 | 原会话值都恢复 |
| Read Committed | 读者先看见 0，确认阻塞写者后，写者提交 1+9 | 拿锁后独立汇总为 10，不能复用旧快照 |
| 真实死锁 | 两事务反向更新 1/2，命名暂停点与 PG 阻塞确认建立环 | 仅一条 40P01，受害者上下文外捕获时已 IDLE；完整重试，尝试数 1/2，最终两行值均 2，效果各一条 |
| 统一锁序 | 两事务按相同键顺序，经守卫登记再 SQL | 有实际等待而无等待环，无死锁，最终同样各生效一次 |

死锁实验故意绕过守卫来观察 PG 检测机制；实验里的两次循环只是原型，不是正式运行时的重试/尝试预算，不覆盖网络错误、副作用或生产提交。

## 4. 验证结果

本地：Python 3.12.12、psycopg 3.3.6、PostgreSQL 16.12（server_version=160012）。

- `make check`：56 个单元/架构用例通过，8 条 import-linter 契约通过，连接器规则和迁移链通过。
- 首轮 `make test-pg-concurrency`：22 个真实 PG 实验通过；随后新增统一锁序双事务实验。
- `TEST_DATABASE_URL=... make test-integration`：最终 24 项通过，其中 PG 并发 23 项、原迁移 1 项。总收集 80 项（56 单元 + 24 集成）。
- 实验后只读查询确认 `m0_lock_probe_` 前缀剩余 schema 数为 0。
- 新增 11 个 Python 文件的 header 与语法、git diff --check 均通过；最终锁序测试采用独立 HLD 期望顺序，不从待测枚举排列反推。

交付 PR：[PR #2](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/2)。最终远端 CI、合并时间和 merge SHA 以该 PR 的 GitHub 记录为准；只在全部检查通过后合并，没有直接向 main 提交。实现提交为 `7e6b538`，后续提交仅关联记录或修复验证发现的问题。

交付确认：PR #2 于 2026-10-30 18:39（Asia/Shanghai）合并，merge=`7b6f76d2f6bcf35219f7a2a35efc15707c15c0cf`。[最终 CI 41042121047](https://github.com/kai2dot0/april_public_sentiment_multi_agent/actions/runs/41042121047) 的 checks / infrastructure 均通过。本地 main 已快进且树与已验证 PR head 一致。下一模块记录见 `2026-10-30-m0-02b-kafka-delivery.md`。

当前复用原 CI 的 `make test-integration` 自动运行新增 PG 用例，无新增工作流或测试框架。Make 增加 `test-pg-concurrency` 用于定向重放；pytest marker 在 pyproject 中登记，未引入依赖变化。

## 5. 核心文件与接手入口

- 设计：`docs/decisions/0002-postgres-concurrency.md`。
- 规则映射：`docs/rule-map/m0-postgres-concurrency.md`。
- 守卫：`runtime/core/lock_order.py`，其单元反例在 `tests/unit/test_lock_order.py`。
- 屏障：`tests/support/failpoints.py`，超时/重复命中验证在 `tests/unit/test_failpoints.py`。
- PG 隔离环境：两级 conftest、`tests/support/postgres.py`。
- 并发实验：`tests/integration/test_postgres_concurrency.py`，不存在随机 sleep 造交错。

复现：先 `make setup`、`make infra-up`；显式配置可丢弃测试库运行 `make test-pg-concurrency` 或完整 `make test-integration`，然后 `make check`。交互 shell 按用户约定加 rtk 前缀；Make/CI 内部不依赖个人 rtk。

后续接手：先阅读本记录和 ADR，按 M0-02 子表完成 Kafka、JCS、CEL、工具链、进程隔离与来源协议验证，再按两批契约引入提交上下文、业务表和锁调用点。M0-05 文件闭环不能把本实验表当作任务/证据表。新增实际写入路径时要补锁序与根来源断言，不以原型测试替代业务并发验收。

## 6. 限制与阶段状态

- 尚未实现 13 步提交、租约 fencing、作业代次变更、删除写屏障、正式重试预算或事务内适配器；不宣告 H1–H17 通过。
- 不增加或更改跨组接口，不迁移/重置业务数据，不修改已有 HLD。
- 静态顺序守卫依赖每个真实加锁调用都先登记且身份规范化；它不会阻止绕过守卫直接执行 SQL。跨组接口冻结和实际接线后需再验收。
- 所有修正继续更新本文件；后续独立技术模块新建 memory 并加入索引。
