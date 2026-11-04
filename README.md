# 市场监管舆情 Multi-Agent 平台

依据 `hld/` 中设计大纲 v1.0 和 HLD v1.2 开发。工程骨架、M0-02 七项技术验证及两批契约已交付，本轮进入 **M0-05A 正式采集判定器**：18 行动作、19 条违规、完整组合验证与签名前重算。任务编排、实际采集、证据业务、模型分析与报告尚未实现，M0 的完整硬验收尚未完成。

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

Python 对象入口拒绝非安全整数；文本入口按 RFC binary64 数值语义解析，可能舍入大数。精确的 64 位 ID 必须用字符串并先经过契约校验。语义包投影与签名现已在第二批实现；证据身份解析和实际回执持久化尚未实现。早期选型与数值域边界见 [ADR 0004](docs/decisions/0004-jcs-canonicalization.md) 和 [开发记录](dev_memory/2026-10-30-m0-02c-jcs-canonicalization.md)。

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

未知枚举保持原始字符串，墓碑载荷经真实 HTTP 往返仍保留；重复键、非法编码和结构损坏明确失败。探针端点仅在测试 app，正式 API 不开放它。正式 FetchPage 已在下述第一批契约提供；签名及独立控制校验见第二批，实际控制提交待运行时接线。

当前维度轮廓全量 3,317,760 个组合，显式排除后扫描 2,488,320 个；耗时只含扫描和未知值检查，不代表完整决策表。方案、数值域发现及 CI 策略见 [ADR 0006](docs/decisions/0006-contract-toolchain.md)、[实测记录](docs/validation/m0-02e-matrix-scan.json) 和 [dev_memory](dev_memory/2026-10-30-m0-02e-contract-toolchain.md)。

## 第一批正式契约

```bash
make test-first-contracts # 模式权限、FetchPage、传输边界、ID 输入与真实 HTTP 往返
make schema-check         # 检查第一批 29、第二批 41 和历史探针 3 份 schema
```

正式模型位于 `contracts/`，接口位于 `spi/transport.py`，SDK 层组合具体消息类型，保持独立叶子边界。接收 Worker 页面必须用 `decode_wire(FetchPageWire, bytes)`，未知状态 / 条目结果和控制载荷原样保留；构造侧才使用严格枚举的 `FetchPage`。所有数字计数限定 JCS 安全范围，外部游标使用字符串；切片导入强制 live / 正式空间 / manual_only，无外部读取。

字段、缺省与调用规范见 [第一批契约 v1](docs/contracts/first-batch-v1.md)、[ADR 0009](docs/decisions/0009-first-contracts.md) 和 [dev_memory](dev_memory/2026-10-31-m0-03-first-contracts.md)。第一批是接口与校验基线，没有生产传输服务或提交持久化；完整包与签名已在下述 M0-04 实现，实际运行时 / SDK / 连接器接线随后开发。

## 第二批正式契约

```bash
make test-second-contracts # 结果包 / 控制 / 状态、摘要签名与事务守卫反例
make schema-check          # 3 + 29 + 41 份 schema，第一批快照保持不变
# 显式专用 TEST_DATABASE_URL 下，make test-integration 包含真实同事务实验
```

`sdk.semantic_bundle` 与平台共享严格字段投影 / JCS / SHA-256；仅排除提交外层 lease_token，稳定身份、控制、剔除和早期段摘要全参与。运行时专用 HMAC 绑定完整任务 / 模式 / 代次 / 配置 / 根 / 控制，SDK 不持密钥。独立逐项控制校验可在未知页面枚举时保留合法控制；材料准备仅检查 V1 / V19，完整决策表、引用和 Claim 载荷验证仍待后续。

同事务上下文明确 I5 / I7 / I9 / I10 / I21 的步骤、根和来源锁；PG facade 只借已有事务，不提供提交 / 回滚或连接工厂。真实 PG 测试证明同事务 identity、提交前不可见、回滚无残留和过期借用失效；业务适配器、13 步实际写入、I8 注册与 I17 / I19 服务尚未实现。

字段、回执优先级、控制对象行与开放项见 [第二批契约 v1](docs/contracts/second-batch-v1.md)、[ADR 0010](docs/decisions/0010-second-contracts.md) 和 [dev_memory](dev_memory/2026-11-03-m0-04-second-contracts.md)。M0-05 分项推进首个文件一页 → 提交 → 检查点 → 事件的真实闭环，当前先完成下面的判定组件。

## 正式采集判定器

```bash
make test-fetch-decisions        # 真实页面、全部动作 / 违规与策略反例
make test-fetch-decision-matrix  # 正式全组合 + gzip 违规明细，默认全量
uv run --locked python -m scripts.verify_fetch_decisions --ordinal 12345 # 重放附件序号
```

规则位于 `contracts/decisions/`，完整 V1–V19 先于 18 行匹配；合法输入必须恰好一行，违规保留全部代码且不执行动作表。平台重新校验控制，未知页面枚举仍保留合法信号；`prepare_collection_material` 在签名前独立重算 PageResult，拒绝伪造动作 / 版本和遗漏违规。

正式矩阵判定 2,764,800 个可达组合，对照独立 oracle；所有动作和违规均覆盖。CI 每 PR 完整运行并保存 `fetch-decision-evidence` 附件 14 天，逐条包含全部违规码。历史 measure-contract-matrix 继续只表示旧规模探针。Schema 检查现在有 3 + 29 + 41 + 2 四组，此前正式快照不变。

调用、来源观测集合、恢复预算 / 提交责任及附件重放见 [采集判定规范](docs/contracts/fetch-decisions-v1.md)、[ADR 0011](docs/decisions/0011-fetch-decisions.md) 和 [dev_memory](dev_memory/2026-11-04-m0-05a-fetch-decisions.md)。本轮输出调度意图，检查点 / 计时器 / 作业状态的真实事务与文件闭环尚待后续。

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
