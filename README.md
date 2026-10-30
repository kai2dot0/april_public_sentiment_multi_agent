# 市场监管舆情 Multi-Agent 平台

依据 `hld/` 中设计大纲 v1.0 和 HLD v1.2 开发。已交付工程骨架与 PG 并发验证，当前增加 **M0-02B Kafka 投递技术验证**：幂等配置、ACK 连续前沿、真实 ACK 丢失与重试实验。任务编排、采集、证据、模型分析与报告尚未实现，M0 的完整硬验收尚未完成。

## 开发启动

前置：Git、Python 3.12（可由 uv 下载）、[uv](https://docs.astral.sh/uv/getting-started/installation/)、Docker Engine / Desktop 与 Docker Compose v2、GNU Make。CI 固定使用 uv 0.9.21 和 Python 3.12。

```bash
make setup          # 安装锁定依赖；仅在 .env 不存在时复制示例
make check          # 不依赖容器的架构、迁移链和单元检查
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
