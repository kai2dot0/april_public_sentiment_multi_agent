# ADR 0001：M0-01 工程骨架

日期：2026-10-30。范围：单仓库开发地基。依据：大纲 §2.4、§9、§10；总册 §2.2、§8；分册一 §0.4、§1；分册二 §0.4、§1。

## 选择与原因

1. Python 3.12 是 HLD 的 3.11+ 候选内选择，本机与 CI 默认统一 3.12；实际补丁版本由安装记录给出。应用与所有模块使用同一环境和锁文件。
2. 使用仓库中按 HLD 命名的独立顶层包，不引入跨语言服务、发布包或额外框架。当前没有可复用业务组件，保留原分层后逐步实现。
3. 使用 uv 0.9.21 管理开发环境与 `uv.lock`。这是新增工程工具，按总册 §11.1 二级选型在本 PR 中记录；不改变公开接口与运行时语义。选择原因：本机已有工具，可锁定传递依赖和下载指定 Python。项目当前 `package=false`，直接从仓库运行，未建立可发布 wheel。
4. FastAPI、Pydantic v2、psycopg 3、Alembic、pytest、httpx、import-linter 均来自 HLD 推荐。只引入现阶段使用的库。SQLAlchemy 是 Alembic 的迁移依赖，不用于应用提交路径；pool、Kafka/Redis 客户端、NLP 与前端依赖待对应模块加入。
5. API 仅提供 `GET /health/live`。不连接数据库，不提供虚假的 readiness、任务或报告成功响应。
6. Alembic 单根单头单链，初始版本只启用 vector 扩展。迁移读 `DATABASE_URL`，只接受 PostgreSQL + psycopg，使用 Read Committed / NullPool；扩展回退不 CASCADE 删除其他数据。具体业务表以契约冻结后的迁移引入。
7. Docker Compose 核心服务 PostgreSQL 16 + pgvector、Kafka KRaft 单节点、Redis。可观测服务通过 profile 按需开启，包含 OTel Collector、Jaeger、Prometheus、Grafana。固定明确镜像 tag；后续更新单独 PR 并复验，不声称当前 tag 是最新版本。
8. 基础服务使用独立宿主端口，全部 loopback。PostgreSQL、Kafka、Grafana 使用命名卷；Redis 是可丢失镜像；Jaeger 只保存开发内存追踪。Kafka 卷挂到镜像中已授权 `/var/lib/kafka/data`，保持镜像默认非 root 用户。
9. import-linter 管项目分层、驱动和连接器直接网络库。其不支持外部子模块规则，`urllib.request` / `http.client` 用小型 AST 补充；允许 `urllib.parse`。这只防显式误用，真实网络执法仍待 worker 进程组实现。

## 验证与后续

- `make check`：锁文件、分层、子模块导入、迁移链和单元反例。
- `make compose-check`：Compose 语法与插值。
- `make test-integration`：真实 PG16 / pgvector / 重复迁移。
- `make smoke-infra`：Redis 与 Kafka 可访问。
- `make smoke-observability`：合成 OTLP span/gauge 实际经 Collector 到 Jaeger / Prometheus；不含来源数据。
- API 工厂仅做启动验证。本阶段不定稿业务契约，不实现假的存储门面、删除屏障、鉴权、限流、进程隔离或模型能力。
- 在无现有代码的条件下，这些是工程选型与初始目录；后续新增的第三方库、契约及事务接口按总册 §11.1 对应级别登记到各自 PR 和 dev_memory。

资料：[uv 项目配置](https://docs.astral.sh/uv/concepts/projects/config/)、[import-linter forbidden](https://import-linter.readthedocs.io/en/stable/contract_types/forbidden/)、[Alembic 教程](https://alembic.sqlalchemy.org/en/latest/tutorial.html)、[Kafka Docker](https://kafka.apache.org/41/getting-started/docker/)、[OTel Collector 配置](https://opentelemetry.io/docs/collector/configuration/)、[Jaeger 2.12 配置](https://www.jaegertracing.io/docs/2.12/deployment/)。这里只引用用法，不以文档代替本项目实测。
