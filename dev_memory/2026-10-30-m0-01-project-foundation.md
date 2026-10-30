# M0-01：工程骨架开发与接手记录

日期：2026-10-30（Asia/Shanghai）。工具：Codex(GPT)。工作分支：`feat/m0-project-skeleton`。

## 1. 背景与授权

项目起始只有 `.git/`、`hld/` 四份已提交设计文档及本机系统文件，没有业务代码。用户要求从 0 按 HLD 实现，当前只启动第一阶段工程骨架；每完成一项功能/模块/设计必须走独立分支 PR 并 merge，禁止直接向主分支提交；每项重要开发保留 dev_memory，后续同功能更新本文件。

本阶段遵循 ponytail 的最小完整实现原则，不省略已触及边界的验证。用户要求所有新代码顶部 AI header（工具、生成日期），函数有注释；未来页面优先复用项目封装组件。仓库暂不存在 `.codegraph/`，未自行建索引，也未委派子 Agent。

## 2. 设计依据与边界

- 大纲 §2.2、§2.4、§9、§10；HLD 总册 §2.2、§7.3、§8、§9；分册一 §0.4；分册二 §0.4、§1。
- 本阶段是 M0 的第一个子单元，不是完整 M0 出口。共享契约、任务/租约、提交/回执、证据/控制表、限流、删除、真实采集、模型网关、报告、React 页面均未实现。
- 空模块只建立 Python 包，不提供返回成功的空业务接口，也不预先冻结字段。
- 首条后续业务闭环以总册 §8.2 集成点 1 为准：一组最小运行时 + 二组 SDK / file / 证据 API + I5/I7/I21 最小真实适配器 + I8 注册 Fake；它不依赖 I9。事务适配器不能用 Fake 替代真实事务。

## 3. 方案与实现入口

### 工程与约束

- 根目录 `pyproject.toml` / `uv.lock` 管共享依赖；`.python-version` 默认 3.12。本机 uv 0.9.21 安装 CPython 3.12.12 到 `.cache/python/`，依赖进 `.venv/`，缓存进 `.cache/uv/`。
- 当前 `package=false`，从单仓源码启动，无 wheel 发布。uv 是新增工程工具，二级选型理由见 `docs/decisions/0001-project-foundation.md`，并通过本功能 PR 记录。
- 按 HLD 建 `contracts`、`spi`、`sdk`、`runtime`、`capabilities`、`adapters`、`agents`、`connectors`、`apps` 包和 `domain-packs` 目录。
- `AGENTS.md` 保存用户约束、PR/merge 与 memory 流程；`.github/pull_request_template.md` 提醒填写规则、测试、迁移及接手信息。
- import-linter 检查叶子包独立、SDK/Agent/连接器依赖、运行时不导插件、worker 侧禁驱动、连接器禁直接网络库。连接器允许经 SDK 间接使用受控传输，因此直接边界与间接依赖区别处理。
- import-linter 实测不支持 `http.client` 等外部子模块规则；`scripts/check_connector_imports.py` 用标准库 AST 补充 `urllib.request` / `http.client`，保留合法 `urllib.parse`。这是静态显式导入检查，不防动态绕过，不替代进程网络策略。

### API

- `apps/api/main.py:create_app` 返回 FastAPI 应用，只开放 `/health/live`（`status=alive`）及文档。
- 导入不连接外部服务，不读取模型密钥，不伪造 `/health/ready`、`/tasks` 或 `/reports`。
- 单元测试用 HLD 已选用的 httpx ASGITransport；没有引入额外测试客户端。

### 数据库迁移

- `alembic.ini` 指向 `adapters/migrations/`，URL 必须从环境读取；仅 PostgreSQL / psycopg。日期按 Asia/Shanghai。
- `env.py` 使用 NullPool 短连接、Read Committed。SQLAlchemy 只用于迁移；运行时提交仍待用 psycopg 显式 SQL 实现。
- 初始版本 `0001_enable_pgvector` 仅 `CREATE EXTENSION IF NOT EXISTS vector`；没有业务表。downgrade 不删除共享扩展，避免 CASCADE 误删其他 schema 数据，只回退版本标记。
- `check_migrations.py` 验证单根、单头、无分叉/合并；添加新业务迁移时更新测试期望版本。
- 模板生成的新迁移保留注释与当次生成时间。离线 SQL 不输出含密码 URL。

### 容器与可观测

- `compose.yaml` 提供 PostgreSQL 16 + pgvector、Kafka 4.1.0（KRaft）、Redis；profile `observability` 提供 Jaeger、OTel Collector、Prometheus、Grafana。
- core 宿主端口为 15432 / 19092 / 16379，全部 loopback；容器 Kafka 地址 `kafka:19092`，宿主 `127.0.0.1:19092`，避免 advertised listener 错配。
- PG / Kafka / Grafana 使用命名卷；Redis 关闭持久化，仅镜像；Jaeger 内存存储。`infra-down` 保留卷，CI 对独立临时环境才清卷。
- 可观测配置在 `deploy/observability/`。没有应用自动埋点和正文采集。`scripts/smoke_observability.py` 发送无内容合成 trace/gauge，验证 Collector → Jaeger、Collector → Prometheus；等待有界、失败明确抛错。
- 目前 Grafana 只有基础数据源，没有虚假的采集/任务/模型指标和业务看板。multixact 与 PG 写入预算将在数据库适配器/真实运行时出现后补。

### CI

- `.github/workflows/ci.yml` 在 PR 与 main push 运行 `checks`、`infrastructure` 两个 job。
- checks：锁文件、导入、迁移链、单元测试、Compose 配置。
- infrastructure：实际启动 PG/Kafka/Redis、迁移、PG 集成测试、Redis/Kafka smoke，再启动可观测 profile 并验证合成信号；失败展示本项目日志，结束清理该 job 临时卷。
- 当前 CI 只验证已交付骨架，不包含未来完整 HLD 门禁。

## 4. 实际遇到的问题与处理

1. 本机初始只有 Python 3.14.7；为统一 HLD 开发版本，使用已有 uv 安装 3.12.12 到项目缓存。
2. sandbox 下 uv 的系统配置读取报错，下载/运行检查通过授权执行环境完成；没有修改个人 uv 安装或全局 Git 配置。
3. Docker Desktop 起初未运行，已启动并实际验证容器。标准 6379 被已有服务占用，改用项目独立高端口，没有停止或修改用户原有服务。
4. Kafka 最初挂载到镜像不存在的 `/tmp/kraft-combined-logs`，新卷 root 所有导致 appuser 无法写 metadata。改用镜像已属 appuser 的 `/var/lib/kafka/data`，以新命名卷验证成功，保持非 root 运行。
5. import-linter 不支持外部子模块约束，采用 AST 补充，相关违规反例进入测试。
6. FastAPI / Starlette 的 TestClient 对原 httpx 有弃用提示，改直接使用 AsyncClient + ASGITransport，未额外引入 httpx2。
7. Alembic 增加 `path_separator=os` 消除 legacy 配置告警；迁移时区固定 Asia/Shanghai。
8. GitHub 连接器及本机默认账号只读；使用本机已登录的仓库所有者 kai2dot0 完成后续 GitHub 操作，不切换用户全局默认账号，令牌不输出、不落库。

## 5. 验证记录

本地已实测：

- `make setup`：成功创建 .env（已存在不覆盖）、.venv 并安装锁定依赖。
- `make compose-check`：全部 profile 配置解析通过。
- `make infra-up`：PG、Redis、Kafka 均 healthy（Kafka 目录修正后）。
- `make migrate`：实际 PostgreSQL 迁移成功。
- `TEST_DATABASE_URL=... make test-integration`：真实 PG16、Read Committed、重复升级和向量距离检查通过。
- `make smoke-infra`：Redis PONG，Kafka 元数据查询成功。
- 四个可观测服务 HTTP 检查均 200。
- `make smoke-observability`：OTel/Grafana health、Jaeger trace export、Prometheus metric scrape 均 passed。
- `make check`：最终 18 个单元与架构反例测试通过、1 个集成测试按 marker 单独运行，无告警；8 条 import-linter 契约全部 kept。
- 临时生成一份 Alembic 迁移：生成 header、Asia/Shanghai 日期和 Python 语法均有效，未在业务迁移目录遗留文件。
- 新增 Python 文件语法及所有新增代码/配置顶部 AI header 检查通过；原 hld 文档未修改。
- 临时随机本机端口真实启动 uvicorn 并请求 `/health/live`：通过；验证后已关闭该临时 API。

远端流程历史（权限阻塞现已解除）：功能分支首个本地提交为 `46a25f7`；最初推送被 GitHub 拒绝，原因是仓库所有者 OAuth 令牌没有 `workflow` scope，无法新增 `.github/workflows/ci.yml`。该次未创建 PR、未运行远端 CI、未合并；没有直接向 main 提交。

已检查现有 SSH 路径：22 端口连接关闭；443 路径缺少已信任的主机密钥，保持 strict 校验，未新增密钥或 known_hosts。已请求用户为 kai2dot0 执行 `gh auth refresh --hostname github.com --scopes workflow`（需先切换至该账号，完成后可切回原默认账号）。授权完成后只需推送现有分支、创建 PR，回填链接并等待 CI 后 merge；不要删除 CI 文件来绕过权限要求。

后续授权复查：用户回复已完成授权后重试推送，GitHub 仍以缺少 workflow 拒绝。通过 `/user` 返回的实际身份和 `X-OAuth-Scopes` 核对，kai2dot0 只有 gist/read:org/repo，kylechandev 有 workflow 但没有目标仓库写权限；已提醒浏览器授权身份也需为 kai2dot0。进一步从 GitHub 官方 HTTPS `/meta` 取得公开 SSH 主机密钥，仅写 `/private/tmp/april-github-known-hosts`，以 strict 校验检查 443 路径；主机验证成功，但现有身份被拒绝（publickey）。没有修改全局 SSH 配置；当时 PR/远端 CI/merge 均未完成。

最终授权恢复：按用户明确要求，将本机 gh 默认账号切换为 kai2dot0，发起 workflow 设备授权，由用户在浏览器完成。CLI 返回 Authentication complete；GitHub `/user` 实测 login=kai2dot0，scopes=gist/read:org/repo/workflow。随后功能分支推送成功，已创建 [PR #1](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/1)。CI 的实际检查、最终合并状态和 merge SHA 以该 PR 的 GitHub 记录为准；只在远端检查通过后执行 PR merge。授权期间无凭据入库、无直接向 main 提交。本次交付仅 M0-01，不代表完整 M0 验收。

## 6. 接手开发顺序

1. 阅读 `README.md`、ADR 0001、`docs/rule-map/m0-foundation.md` 和 HLD 总册 §3 / §7.4 / §8。
2. `make setup && make check && make compose-check`（在交互式 shell 按用户约定用 rtk 前缀分别执行）；启动容器后迁移和集成测试。
3. 按 `docs/roadmap.md` 进入技术验证与分批契约冻结，先做 PG 双连接锁/屏障、JCS、Kafka producer、进程隔离等验证。不要把 API 存活等同于平台 ready。
4. 冻结契约后再建任务、控制对象、证据等业务表，执行 migration-check；为新约束更新规则—实现—夹具映射。
5. 补 worker 的注册、运行隔离和真实删除能力；所有内容存储必须有写屏障与删除处理器，不能先提供假成功实现。
6. M1 再建设真实来源、网关与分析链、报告及共享 React 组件；届时复用组件风格。

## 7. 限制与当前环境

- 未验证全部 HLD 硬判据、完整业务可靠性、真实 Jetstream/AppView、真实模型、负载与删除闭环；这些均在后续单元。
- `TEST_DATABASE_URL` 必须是可丢弃专用库；当前测试会升级指定库，但不执行 downgrade 或删除业务数据。
- 本机 smoke 后 7 个开发容器保留运行；临时 API 进程已退出。需要停止时执行 `make infra-down`，保留数据卷。本阶段不改变仓库 main 的保护设置。
- 后续同功能修复请更新本文件；新模块新建自己的记录并补索引。
