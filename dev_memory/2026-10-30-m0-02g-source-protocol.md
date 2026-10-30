# M0-02G：来源协议、游标、缓存与许可核实

开发始于 2026-10-30 23:54，跨至 2026-10-31（Asia/Shanghai）；文件名沿用开工日期。用户授权“继续下一项”，按 roadmap 推进 G；没有开始 M0-03。所有代码在功能分支 `feat/m0-source-protocol-verification`，经 PR / 实际 CI / merge 交付。

## 1. 背景与范围

用户要求依 HLD 从空工程完整开发，每模块 PR 合并、保留可接手 memory。本轮依据分册二 §0.5.2、§7、§8、§16，大纲 §8.3 / §4.12 与总册 §11.1 / §11.4：必须先核实具体协议和来源事实，不能把设计中列举的错误、时间或许可当作已证明的上游保证。

已实现：固定来源证据登记、明确 v2 候选和两代差异、离线控制元数据分类、真实 loopback WS 断线与 inclusive 恢复、账户和 sync 保守分类、AppView 缺项 / 缓存反例、手动元数据脚本及一次实际公网实验。记录许可现状和开放条件；**没有宣布来源合规已批准，也没有开始真实采集**。

不在本轮：正式 ConnectorManifest、FetchPage、I4 受控传输、首次资格、持久接收 / 提交 / 检查点、账户权威代次、删除写入、resync、完整 S2、RSS 解析器、第三方模型处理。正式业务仍无来源正文入库。

## 2. 调查方案和可复核依据

1. 读根 AGENTS / RTK / ponytail 与 HLD；确认 `.codegraph/` 不存在，不建索引；main 清洁且已包含 PR #7。
2. 官方 web 资料发现 Jetstream 主仓库已重构，原代码迁 jetstream-legacy。用 GitHub 只读 API 取 SHA，raw 固定提交读取 Lexicon / cursor / encoder / 账户定义，不执行资料内自建 relay 的命令。
3. 将 URL / SHA-256 放 source-lock，不复制文档全文、真实示例 DID / 文本；公开政策只保存阅读结论。TOS 直接取字节遇 403，经 web 工具阅读，明确不声称字节快照。
4. 先离线构造虚构帧，再用固定官方端点和虚构 DID / URI 测握手 / 控制提示 / 数量边界；不读取实际帖子流，v1 requireHello 暂停后只握手。
5. 将调查证据、能力未知与项目停止政策分开，疑问登记 G-01–08；没有访问 Confluence 或冒充负责人 / 法务签字。

源码锁：Jetstream `c9cbd1b3e6604b27bcb32cad08801b9806d4705a`；legacy `8a65de4eda28bed1cafcbcf25b0cd46ac6f2148b`；atproto `a2dd244d6072216735198db5777909d014b0638e`；bps-website `8a8a57d9df74b7a1eccbc58ac778609422da1e14`。公网实际部署 commit unknown，不能用这四个 SHA 冒充部署身份。

## 3. 文件与分层

| 文件 | 用途 / 核心点 |
| --- | --- |
| `tests/support/source_protocol_probe.py` | 实验性 V1Frame / V2Frame、ProbeCursor、Observation、HTTP / posts 分类；只在 tests / script 使用，不作为正式业务 parser |
| `tests/unit/test_source_protocol.py` | 协议错误、账户状态、跨实例反例、真实 WS、缓存和元数据脱敏回归 |
| `scripts/probe_source_metadata.py` | 显式手动四次固定 WS / 两次 HTTP；无任意 URL CLI，虚构 DID / URI，输出白名单状态 / 计数 |
| `docs/validation/m0-02g-source-lock.json` | 10 个固定源码 / 文档文件摘要、网页政策来源、unknown 项、禁用开关、RSS 预查 |
| `docs/validation/m0-02g-live-metadata.json` | 本次实际公网脚本输出的元数据，不是固定 CI 期望值 |
| `docs/decisions/0008-source-protocol-verification.md` | 完整证据、差异与保守政策、许可边界和选型 |
| `docs/rule-map/m0-source-protocol.md` | 规则映射与 §11.4 语义的仓库登记表，区分调查落实和仍开放问题 |
| `pyproject.toml` / `uv.lock` / `Makefile` | websockets 开发依赖、离线 / 手动入口；默认 check 自动收集离线用例 |

复用 WireModel / decode_wire 严格编码与重复键检查，复用 AckPrefix 验证本地接收前缀，不新造一套 JSON 解析器或 ACK 状态机。frame 内 dict 只检查本轮所需的控制元数据，**没有完整校验 DID / rev / CID / record / time 等 Lexicon 字段**；生产接入时需要正式结构和预算，不应直接提升此测试 helper。

新增开发依赖 websockets 17.4（BSD-3-Clause，声明 >=17.4,<18，uv.lock 精确锁），用途是标准 WS 实现替代手写 framing。无生产依赖、contracts / spi、事务、锁序或迁移变化；按总册 §11.1 留二级组件实施记录，无 A/B/C 裁剪。三个新 Python 顶部 AI header / 函数说明，JSON 无注释；原文件不补 header。

## 4. 核心语义与实现细节

- 候选 v2 east / west 的完整 URL 在 ENDPOINTS；v1 east2 / west1 仅证据。ProbeCursor 绑定 version + endpoint，恢复必须完全相同。v2 要求 `0 < seq < 10^15`，v1 实验支持 `10^15 <= time_us < 2^63`；拒绝 bool、str、负数和时间 / seq 混用，恢复原值不加一。
- v2 外层 `$type=message`，payload `$type=NSID#kind`；terminal 为 `$type=error`。info 无 seq，OutdatedCursor → cursor_expired，FutureCursor → invalid_job，未知 info → transport_error；全部不产生可确认位置。HTTP 400 CursorTooOld 单独映射过期；同名终止帧或 503 不冒充该升级前错误。
- ordinary Observation.position 是已观察位置，不是数据库检查点。seq 从外层 Jetstream payload 取，绝不取嵌套 relay.seq。过滤产生 11→99 这种跳跃不等于内部因果缺口。
- account.active=false 不是永久删除布尔值；deleted 只给候选，已知其他 inactive 状态暂不可用、缺失 / 未知 / 矛盾状态 unverified。sync_unverified 不确认复活、完整性或清扫完成。
- observe_posts 接受 1–25 个唯一请求地址，拒绝响应集合外地址和重复地址；缺项 unresolved，freshness_lower_bound 固定 unknown，deletion_confirmed=false。即使 Date / Age / max-age 都在，也不推导 AppView 索引进度。
- 探针帧上限 64 KiB 仅为实验预算。真实来源生产上限和超限语义未来冻结，不能靠本预算跳过真实事件后仍确认前沿。
- WS 夹具随机 loopback 端口，真实握手 / close / reconnect；服务线程异常回传、有界 shutdown / join，无依赖随机 sleep。新用例默认不访问公网。

## 5. 公网实际结果与许可现状

脚本实际执行 UTC 2026-10-30 15:59:37–15:59:42（上海 23:59），只保留元数据：

| 实验 | 结果 |
| --- | --- |
| v2 east cursor=1 | 升级前 400，error=CursorTooOld |
| v2 east cursor=999999999999999 | 101，xrpc.v1.json，首提示映射 invalid_job，无位置 |
| v2 west cursor=1000000000000000 | 101，xrpc.v1.json，首提示映射 cursor_expired，无位置 |
| v1 east2 requireHello=true | 101，无 subprotocol；未释放流、未读实际事件 |
| public AppView 25 个虚构 URI | 200，posts=[]，unresolved=25；max-age=30，freshness unknown |
| 同端点 26 个虚构 URI | 400，max-age=5；不进行额外探测或限流压测 |

RSS 附带预查 UTC 16:01：stats.gov.cn 两个 HLD 示例 feed HEAD 200 / text/xml，robots GET 404；服务条款包含注明来源及第三方 / 禁转载等排除。没有 feed 正文下载 / 入库，没有以 robots 404 代替用途授权。

Jetstream 软件 MIT / Apache-2.0 许可不代表帖子数据授权。Bluesky 开发者指南要求删除机制等；TOS 用户内容许可不能自动当成本项目的任意模型处理 / 展示 / 分发权。当前具体保留期限、第三方模型处理、展示摘录和删除时限资格未落实，source-lock collection=false。G-06 / G-07 / G-08 仍是后续真实启用前置，技术核实交付不表示这些资格已关闭。

## 6. 验证及修正记录

初轮 41 项协议测试中 40 通过，1 项因 source-lock 写入遇 TOS 403、文件未生成而失败。修正为将固定源码字节摘要和仅经 web 阅读的政策依据分列，不捏造政策快照；随后 41/41 通过。进一步补 operation 非字符串反例及包含 Date / Age / max-age 的 AppView 离线响应测试。

第一轮 make check：8 项导入契约、连接器静态检查、1 个迁移版本、3 份 schema 均通过；394 个离线测试通过，39 个真实基础设施用例按标记交给独立任务。补充 operation、缓存响应和未协商二进制帧反例后，专用协议套件 44/44 通过；最终 make check 为 **397 passed / 39 deselected（10.78 秒）**，所有工程门禁通过。三个新 Python 的 header / 函数说明 / 语法、两份 JSON 和 git diff --check 均有效。公网一次实验通过取得上表事实，不能泛化成服务 SLA。

```bash
rtk proxy make check
rtk proxy make test-source-protocol
# 以下只显式手动执行，不属于默认 CI：
rtk proxy make probe-source-metadata
```

三项远端 CI checks / infrastructure / isolation 保持原有职责；本轮不重复改动隔离 / PG / Kafka 实现，合并前必须确认最终 head 的三项全部成功。实际 PR / 最终验证结果在交付时补记，不预填通过。

## 7. 接手顺序与后续边界

1. 读 ADR 0008、G 登记表、两份 validation JSON，理解 source evidence / measured fact / unknown 的区别。
2. 看 helper 与 test，明确它们是控制元数据实验，不是正式完整 parser 或业务服务。
3. 下一项 M0-03 冻结第一批契约。正式来源 Manifest 后续要体现实例身份、协议版本、cursor domain、retention、resume、caught-up 和控制投递保证，不能删除 unknown 来伪装具备首次资格。
4. I4 接入后所有真实请求经受控传输 / 代理 / 预算；不要让 connector 导入 websockets / httpx 或调用本手动脚本。
5. M1 真实来源前完成 S2 公网断线 / 恢复 / 跨实例 handoff 夹具、权威作业和接收登记、持久连续前缀、账户 / sync / 删除全链路。若仍无有界投递或 freshness 依据，沿 HLD 保守路径，不能启用接收时间证明或缺项删除推断。
6. 所有来源按实际用途登记来源 / 权限 / 保留 / 模型 / 展示 / 删除 / 核实依据与时效；原始 HLD 文档未被静默改写。

回退无数据 / schema 迁移：移除测试 helper、脚本、Make 入口和开发依赖即可；保留必要证据用于回顾。无需清理现有 Docker 服务 / 卷，脚本没有持久 WS 会话。

## 8. PR 与交付

本模块 [PR #8](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/8)，实现提交 `86bcbf1`；本次文档补记也进入同一 PR，必须等待其最终 head 的 checks / infrastructure / isolation 三项全部成功才合并。最终受检提交、CI run、merge SHA / 状态以 PR 实际记录为准，不把先前 head 的绿灯移用到后续提交。上一模块 PR #7 合并证据已补原 F memory；本模块不直接推 main。
