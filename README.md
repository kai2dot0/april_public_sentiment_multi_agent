# 市场监管舆情 Multi-Agent 平台

依据 `hld/` 中设计大纲 v1.0 和 HLD v1.2 开发。已交付工程骨架及 PG / Kafka / JCS / CEL / 契约工具链验证，当前增加 **M0-02F worker 隔离验证**：真实 cgroup OOM、角色网络策略、整树终止和缓存清理核验。任务编排、采集、证据、模型分析与报告尚未实现，M0 的完整硬验收尚未完成。

## 开发启动

前置：Git、Python 3.12（可由 uv 下载）、[uv](https://docs.astral.sh/uv/getting-started/installation/)、Docker Engine / Desktop 与 Docker Compose v2、GNU Make。CI 固定使用 uv 0.9.21 和 Python 3.12。

```bash
make setup          # 安装锁定依赖；仅在 .env 不存在时复制示例
make check          # 不依赖容器的架构、迁移链、schema 和单元检查
make compose-check  # 检查全部 Compose 配置，不启动容器
make infra-up       # 启动 PostgreSQL 16 + pgvector、Kafka、Redis，等待健康
make migrate        # 对 .env 中 DATABASE_URL 执行迁移
make smoke-infra    # Redis PING 与 Kafka 元数据访问
make api            # 启动本机 HTTP API
```

访问 `http://127.0.0.1:8000/health/live` 返回 `{"status":"alive"}`，`/docs` 提供接口文档。存活检查只证明 API 进程可响应；没有业务 readiness 接口，不代表任务、模型或数据库处理就绪。

`.env` 不入库。示例凭据仅供本机开发，禁止用于共享或生产部署。依赖缓存和 uv 管理的 Python 保存在 `.cache/`，虚拟环境位于 `.venv/`，均不入库。

## 真实数据库验证

仅指向本项目专用、可丢弃的开发测试库，禁止使用生产库或保存有价值数据的数据库：

```bash
TEST_DATABASE_URL='postgresql://april:april_local_only@127.0.0.1:15432/april' make test-integration
```

集成测试实际执行迁移并检查重复升级、PostgreSQL 16、Read Committed、pgvector 运算。缺少 `TEST_DATABASE_URL` 时明确失败，不自动跳过。

同一套集成命令还会执行 PG 并发实验：四种行锁 16 对、SKIP LOCKED、事务局部配置、等锁后重读、真实死锁/完整回滚重试和统一锁序。每个实验使用独立随机 schema，关闭连接后仅清理自己创建的命名空间。仅重放该模块时可运行 `TEST_DATABASE_URL='...' make test-pg-concurrency`。方案与边界见 [ADR 0002](docs/decisions/0002-postgres-concurrency.md) 和 [开发记录](dev_memory/2026-10-30-m0-02a-postgres-concurrency.md)；守卫不是正式提交或删除协议的替代实现。

迁移 URL 由环境变量提供，接受 `postgresql://` 和 `postgresql+psycopg://`。SQLAlchemy 只用于 Alembic；未来运行时的提交路径使用 psycopg 显式 SQL。初始迁移仅启用 vector 扩展，不提前定义任务、证据或控制对象表。

## Kafka 投递验证

只对本项目可丢弃开发 broker 执行。故障拓扑会重启本项目 Kafka，临时把 HOST advertised 地址指向测试进程的 loopback 代理；测试完成后恢复普通拓扑，保留卷：

```bash
make kafka-test-up
TEST_KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:19092 TEST_KAFKA_PROXY_PORT=19093 make test-kafka
make infra-up
```

测试用随机 topic 和合成消息验证：正常回调、broker 已写入后的 ACK 丢失、持续丢 ACK 的歧义超时、新生产者重发重复。客户端内部重试在幂等模式下去重，不代表应用按事件身份去重已实现；flush 剩余为 0 也不代表全部投递成功。方案见 [ADR 0003](docs/decisions/0003-kafka-delivery.md) 与 [开发记录](dev_memory/2026-10-30-m0-02b-kafka-delivery.md)。

`TEST_KAFKA_BOOTSTRAP_SERVERS` 必须显式提供，不回退业务配置；代理端口默认 19093，修改 `.env` 中 `TEST_KAFKA_PROXY_PORT` 后测试进程也须导出同值。不并行运行共用相同 broker/端口的故障套件。原 `make test-integration` 仍运行数据库集成，Kafka 单独运行；CI 包含两者。

## JCS 规范化验证

```bash
make test-jcs       # 离线重放 RFC 向量与输入边界；也包含在 make check 中
```

锁定 `rfc8785 0.1.4`，内部原型输出 JCS UTF-8 字节与完整输入值的 SHA-256。对象键递归按 UTF-16 排序，数组顺序和 Unicode 形式保留；严格文本入口拒绝重复键、非法 Unicode、非有限数值与非 UTF-8 编码。

Python 对象入口拒绝非安全整数；文本入口按 RFC binary64 数值语义解析，可能舍入大数。精确的 64 位 ID 必须用字符串并先经过契约校验。当前未实现语义包字段投影、签名、证据身份或回执。方案与数值域边界见 [ADR 0004](docs/decisions/0004-jcs-canonicalization.md) 和 [开发记录](dev_memory/2026-10-30-m0-02c-jcs-canonicalization.md)。

## CEL 规则验证

```bash
make test-cel       # 离线运行子集、准入与真实子进程超时测试；也包含在 make check
```

内部 `evaluate_rule(expression, facts)` 使用锁定的 `common-expression-language 0.10.0`。表达式通过 stdin 进入独立进程，求值前检查完整函数引用；只返回 bool，语法、类型和超时错误单独报错。现阶段限制 4096 字节表达式、每输入集合 128 项和 64 KiB 请求，缺省 2 秒截止时间，超时 kill 并回收。

本轮选择 HLD 允许的超时方案；没有精确操作成本或硬内存上限，也未完成 OS 沙箱。候选比较、完整预算与边界见 [ADR 0005](docs/decisions/0005-cel-subset.md) 和 [开发记录](dev_memory/2026-10-30-m0-02d-cel-subset.md)。领域包规则表、执行配置锁和 risk-grader 按路线图在 M1 接入。

## 契约工具链验证

```bash
make test-contract-toolchain  # schema、真实 loopback HTTP/JSON、未知枚举与规模探针
make schema-check            # 只读检查模型导出与快照，不自动改写
make measure-contract-matrix # 全量维度扫描和耗时；不是 H11 业务验收
```

采用 HTTP/JSON，复用既有依赖。`contracts/tooling.py` 提供严格字节解码与字段元数据原型；测试模型只导出到 `contracts/schemas/probes/`，`make check` 比对漂移。修改模型后可显式 `make schema-write`，审查生成 diff 后通过 PR。

未知枚举保持原始字符串，墓碑载荷经真实 HTTP 往返仍保留；重复键、非法编码和结构损坏明确失败。探针端点仅在测试 app，正式 API 不开放它。完整 FetchPage、控制提交和签名按后续阶段实现。

当前维度轮廓全量 3,317,760 个组合，显式排除后扫描 2,488,320 个；耗时只含扫描和未知值检查，不代表完整决策表。方案、数值域发现及 CI 策略见 [ADR 0006](docs/decisions/0006-contract-toolchain.md)、[实测记录](docs/validation/m0-02e-matrix-scan.json) 和 [dev_memory](dev_memory/2026-10-30-m0-02e-contract-toolchain.md)。

## worker 隔离验证

只对专用、可丢弃的开发 Docker Engine 执行；要求 Linux cgroup v2（Docker Desktop 使用其 Linux VM），会创建独立合成容器 / 卷 / internal 网络，不挂载现有项目卷，也不改宿主防火墙：

```bash
make isolation-build
TEST_ISOLATION_ENABLED=1 make test-isolation
```

采用每 worker 独立受限容器的替代原型，可信控制端通过 Docker 管理其 cgroup；worker 无 Docker socket / 数据库凭据。一次性 NET_ADMIN helper 仅在该 worker 私有 netns 安装规则，host / 非内部网络在准入时拒绝。真实测试包含 OOM、忽略 TERM / setsid 后代终止、活端点网络正反例、只读根 / tmpfs 与缓存不可核对负例。清理只针对本次所有者标签匹配的资源，不用全局 prune。

缺少开关、cgroup v2 或所需内核能力时明确失败，不跳过。`make test-integration` 仍只跑 PG，隔离套件单独执行并由专用 CI job 验证。替代方案、端口表和控制面边界见 [ADR 0007](docs/decisions/0007-worker-isolation.md) 与 [dev_memory](dev_memory/2026-10-30-m0-02f-worker-isolation.md)。本轮输出仅为本地清理证据，不代表登记服务退役、fencing 或真实任务重派已实现。

## 来源协议核实

```bash
make test-source-protocol   # 离线回归，真实 loopback WebSocket；已纳入 make check
make probe-source-metadata # 显式访问固定官方端点，只用虚构 DID / URI，输出状态与计数
```

后续来源开发选 Jetstream v2 的实例内 seq；v1 时间游标、嵌套 relay.seq 和其他实例不能混用。探针验证旧游标 / 未来游标的停止映射、inclusive 重复恢复、account / sync 分类、AppView 25 URI 上限与缺项处理。它不是正式连接器或受控传输服务；默认 CI 不访问公网。

公网实测、固定源码摘要和来源许可边界见 [ADR 0008](docs/decisions/0008-source-protocol-verification.md)、[核实登记表](docs/rule-map/m0-source-protocol.md) 与 [dev_memory](dev_memory/2026-10-30-m0-02g-source-protocol.md)。控制事件投递上界、caught-up 证明和 AppView 索引新鲜度下界仍 unknown；接收时间证明关闭，真实采集未启用，公开接口 / 软件许可证不代替内容和模型用途许可。

## 可观测开发环境

```bash
make observability-up
make smoke-observability
```

| 服务 | 本机地址 | 说明 |
| --- | --- | --- |
| PostgreSQL | `127.0.0.1:15432` | 数据与迁移；开发用户有创建扩展权限 |
| Kafka | `127.0.0.1:19092` | KRaft 单节点；容器内地址 `kafka:19092` |
| Redis | `127.0.0.1:16379` | 可丢失镜像，关闭持久化 |
| Jaeger | `http://127.0.0.1:16686` | 内存存储的开发追踪，重启会丢失 |
| OTel Collector | `127.0.0.1:4317` / `4318` | OTLP gRPC / HTTP；health 在 `13133` |
| Prometheus | `http://127.0.0.1:9090` | 基础监控与导出的合成指标 |
| Grafana | `http://127.0.0.1:3000` | 账号见 `.env`；已配置 Prometheus、Jaeger 数据源 |

全部公开端口绑定 loopback。基础服务端口可在 `.env` 调整，数据库端口或凭据变化时同步修改 `DATABASE_URL`。当前 Compose 是本机基础设施环境，尚未部署 Agent/worker，也未实现它们的默认拒绝出站、cgroup、撤回缓存等控制。

`smoke-observability` 仅发送无来源内容的合成 span 和 gauge，再从 Jaeger 和 Prometheus 查询确认。当前没有应用自动埋点、请求体、SQL 参数或来源正文采集；后续业务埋点必须满足 HLD 的无原文要求。

```bash
make infra-down     # 停止全部开发服务，保留数据库、Kafka、Grafana 数据卷
```

不要在日常停止时增加 `--volumes`；CI 只对该次运行创建的临时环境清卷。不要在宿主机和容器间直接共享数据库文件。

## 工程目录

| 路径 | 职责 |
| --- | --- |
| `contracts/`、`spi/` | 独立共享契约与协议；具体定义待冻结 |
| `sdk/` | Worker / Agent / 连接器客户端能力 |
| `runtime/` | 通用任务与提交运行时 |
| `capabilities/` | 证据、知识、网关、事务内适配器等平台能力 |
| `adapters/` | 基础设施实现；当前含 Alembic 迁移 |
| `agents/`、`connectors/` | 分析与采集插件 |
| `domain-packs/` | 发布期领域声明；本阶段仅保留说明 |
| `apps/api/`、`apps/console/` | API 装配与后续 Web 控制台 |
| `deploy/` | 开发基础设施配置 |
| `tests/`、`scripts/` | 现阶段验证与门禁 |
| `docs/`、`dev_memory/` | 阶段计划、规则映射、选型与接手记录 |

## 协作与接手

先读 [开发约定](AGENTS.md)、[阶段计划](docs/roadmap.md)、[工程设计](docs/decisions/0001-project-foundation.md)、[规则映射](docs/rule-map/m0-foundation.md) 和 [本阶段开发记录](dev_memory/2026-10-30-m0-01-project-foundation.md)。

每项功能或重要设计走独立分支 → 验证 → 更新对应 `dev_memory` → PR → merge。禁止直接向主分支提交。CI 运行本地门禁和真实容器检查；后续模块按 HLD 逐步补充契约、锁序、删除及故障验收，不能把当前 CI 全绿当作 M0 全部通过。
