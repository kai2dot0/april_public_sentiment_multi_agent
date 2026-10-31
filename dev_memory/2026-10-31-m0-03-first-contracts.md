# M0-03：第一批正式契约工程基线

日期：2026-10-31（Asia/Shanghai），工具 Codex(GPT)。起点 main `340db0fcd7c91321c02d80ce823dcaea3706ed9c`，PR #8 已合并；分支 `feat/m0-first-contracts`。用户要求“继续下一项开发”，按 roadmap 的 M0-03 交付，不开始第二批或业务运行时。

## 1. 依据、目标与授权范围

总册 §3 / §8.2 第一批：执行上下文、受控传输、FetchPage / item_outcomes、状态与错误映射、ID 派生输入。分册一 §1 要求 Pydantic 唯一模型、字段元数据、开放线枚举 / 严格构造分层；分册二 §2 / §3 / §6 要求传输绑定上下文与预算、准入 / 接收登记 / 运维指定信任边界。大纲 §4.6、§4.7、§4.9、§4.11、§8.2 是语义依据。

本轮把草案变成可校验、可导出、可由 SDK / 后续服务引用的正式工程定义：29 个公开根模型、泛型 SPI 和 SDK 类型绑定、模式 / 计划纯校验、状态处理表、ID 输入材料与兼容反例。新跨组接口按三级影响在 ADR 0009 / C03-01–12 登记；没有旧业务消费者 / 数据，不涉及迁移或返工，没有 A/B/C 裁剪。

用户的模块 PR → 检查 → merge 流程持续适用；HLD 的组织角色签字没有被模拟，不声称双方接口负责人已签字。没有 .codegraph/，没有索引或子 Agent；所有 shell 调用用 RTK。使用 ponytail，复用已有严格 JSON、Schema 元数据、API 工厂和 HTTP 生命周期，没有新增依赖。

## 2. 设计方案和落地顺序

1. 对照总册冻结清单与两分册，确认 18 / 19 YAML 的 deadline 是 M0 内、完整提交在第二批，不把它们混入第一批字段冻结。
2. 把公共数字 / 时间 / 引用 / 位置域建在 `contracts.primitives`；构造模型继承已有 WireModel，集中声明字段元数据，动态 JSON 也做安全整数检查。
3. 建 execution / fetch / transport / states / identities 模型；FetchPageWire 的未知枚举保留与控制载荷保留在同一解码路径，构造侧另用严格模型。
4. 保持 contracts 和 spi 独立叶子：SPI 用标准库泛型 Protocol，SDK 层 type alias 绑定具体消息。避免为具体参数类型放宽现有 import-linter。
5. 把模型注册到唯一 registry，从代码生成 v1 Schema；原探针独立路径和默认工具 API 不改义。普通检查只读，显式 write 才生成。
6. 补权限、数值、伪造时间、错误处理、ID 材料和真实 HTTP 反例；把已有 loopback 生命周期提取为共享测试 helper，保留原探针语义。
7. 更新规范、ADR、映射、README、roadmap 和本 memory；完整检查通过后提交独立 PR，等最终 head 三项 CI 成功再合并。

## 3. 核心文件与接手定位

| 文件 | 内容 |
| --- | --- |
| `contracts/primitives.py` | ContractModel、contract_field、安全整数、原样标识符 / UTC、StableReference、PositionDomain / SourcePosition、Checkpoint |
| `contracts/execution.py` | 上下文、主体、空间、显式授权 / 配额、计划 / 能力及 Demo 准入纯函数 |
| `contracts/fetch.py` | 完整页字段、RawItem / 条目结果、ObservationEvidence、Coverage 与两种报告、开放 / 严格模型、可信判定上下文、V1 路径定位 |
| `contracts/transport.py` | HTTP / WS 消息、绑定 / 预算、准入、冷却、实时连接、公开与可信接收登记、中断审计、运维指定 |
| `contracts/states.py` | 唯一状态域、冲突 / retry / submission / fail 的处理语义；没有创建提交回执模型 |
| `contracts/identities.py` | 四元组输入和稳定字节材料；不在本轮实现 SDK 哈希 / 对象 ID 输出 |
| `contracts/registry.py` | 29 个公开根模型注册表，嵌套模型通过 schema $defs 导出 |
| `contracts/schemas/v1/` | 29 份生成快照（约 112 KB）与消费说明；原 3 份 probes 不改变 |
| `spi/transport.py` | ControlledTransport / ControlledSocket、RateAdmission、RealtimeRegistry、AuthorityRegistry 泛型行为接口 |
| `sdk/transport_types.py` | 两个叶子在 SDK 层组合；TransportHandle / WebSocketHandle 等具体别名 |
| `scripts/check_contract_schemas.py` | 同时检查 v1 / probes；schema_documents / check_schemas 的缺省仍为 probes，兼容既有测试调用 |
| `tests/support/http_server.py` | 从既有测试提取的实际 TCP loopback、启动事件和有界回收 |
| `tests/unit/test_first_contracts.py` | 81 项正式契约测试；原 test_contract_toolchain 的 43 项继续回归 |
| `docs/contracts/first-batch-v1.md` | 调用方规范：字段与默认、权限、I4、状态错误、ID 字节格式及限制 |
| `docs/decisions/0009-first-contracts.md` / `docs/rule-map/m0-first-contracts.md` | 方案、分层理由、三级影响、HLD 映射与待后续落实事项 |

11 个新 Python 文件顶端均有 AI header / Codex(GPT) / 本地日期，所有新增函数有 docstring；修改已有文件不补 header。JSON 快照不加注释。没有修改 pyproject / uv.lock / Docker / CI 拓扑或数据库迁移。

## 4. 关键契约决策

### 数值、编码与默认

- 数字计数 / 代次为 `0..2^53-1`；生成段号 / 指定版本从 1 起。大于安全范围的源 seq / 计数需用字符串，不能先转 float；动态 JSON 检查点 / 控制载荷也拒绝宽整数。未改变 M0-02E 的 int64 历史探针。
- UTC 使用合法日历的 Z 文本、可保留 1–6 位微秒，不规范化成其他精度 / 时区。Identifier 拒绝空白、C0 / C1 控制字符和代理项，不 trim / NFC。
- 所有本批声明字段 digest=include；默认值展开有元数据，必填 nullable 的两个 checkpoint 不能直接省略。不是摘要字段投影，也没有 lease token 排除或签名。
- 固定属性不是完整深冻结，嵌套 JSON / list 可变；平台不能因此信任 Worker。后续真实入口需重新严格解码与认证，禁止 model_construct 绕过。

### 模式和计划

live / backfill 写 formal；replay / shadow / eval 写 isolated，不能重新采集或产生外部副作用，仅其他读取能力可用显式 canary。backfill 无外部读取，不运行 collector。切片导入固定 live / formal / manual_only / 无外部读取，source_namespace 保留；切片复现 replay / isolated。

能力 / 工具 / 池必须显式，允许池与配额恰好对应且无重复。外部读取授权必须属于允许能力，connector grant 绑定 job。上下文 JSON 的 caller.role 不等同于已认证服务身份。

计划校验拒绝 Demo long_session、无恢复的 continuous bounded_slice、未满足 required continuity 的能力；history 必须 manual_only；真实来源配置即使改成 guarantee 也会拒绝，只有 http-fixture / test 可用。这个纯函数不替代未来 Manifest 解析 / 有效配置锁 / 凭据授权。

### FetchPage 与控制保留

线模型涵盖 HLD 全部页字段，枚举用原样 str。未知 status / completeness / end_reason / failure_scope / outcome / forbidden reason 可安全解码并保留 tombstones / control_events。unknown_fetch_enum_paths 只定位 V1，缺 forbidden 原因这种 V2 等语义组合仍由后续判定器分类。

构造侧 FetchPage / ItemResult 用已知 Literal，不能拿它们当 Worker 请求模型；否则会在未知值时丢控制载荷。控制数组本批为保留内容的 JSON 对象：独立控制结构 / 权限校验和 V19 在第二批及运行时中落实，解码成功不表示任意控制可写。

RawItem 关联 item_ref 与稳定原始引用，不携带 blob instance。观测事实分别保留新鲜性下界 / 存在上界 / 门控代次 / 依据，均可未知，不能从 observed_at 自动补造。位置域含 source / instance / ordering domain / cursor semantics；值字符串。两类交接报告可重复携带，起点未知保留 null，不使用首事件替代 accepted resume 起点。

FetchDecisionContext 来自平台可靠登记，不信任 Worker 的观测集合 / 控制校验结果；单资源 × 查询或流不可达，构造层与后续扫描排除要一致。完整 V1–V19 / 18 动作表没有在此实现，不能声称 H11 通过。

### 传输与信任边界

I4 只定义消息和接口。TransportBinding 的非空 grant 必须已经在上下文中；file / 切片导入使用显式 grant=null 且 origins / methods 都为空的无网句柄，不为统一签名虚构外部权限。io 请求 / 解压字节 / 墙钟预算不能扩大；消息数 / 页数由显式切片预算给出。origin 精确，不按 URL 前缀授权；URL / header 只做结构检查，不代替网络执法、DNS、TLS 或重定向策略。

HTTP / WS 字节用标准 canonical Base64，重复头按有序数组保留；文本 WS 是有效 UTF-8。connector 请求不允许覆盖 Authorization / Cookie / Host / Upgrade 等保留头。没有实际凭据在模型 / 日志中。

公开 record_receipt(frame_ref, upstream_event_identity) 不接受任何时间 / catch-up / connection / seq 输入。TrustedReceiptRegistration 仅可信 SDK → 服务使用，必须经服务身份与帧记录核对；不能因 Python 可 import 到类型就给插件发送该消息的权利。recorded 必须有 durable earliest metadata；unregistered 没有任何接收时间证明。连接登记在订阅前，authority 返回只是快照，提交仍需 I7 即时查询；失败不中断既有流的实际实现和两分钟故障夹具留到后续。

SPI 不能直接实例化，没有生产 Fake，也没有默认成功态。当前 SDK 只是具体类型别名，不是网络实现 / Worker 循环。已登记 / 准入的模型形状检查不能证明 PG 持久化或实际许可消耗。

### 状态 / 错误和 ID

五种冲突原因、两种 retry 原因、submission 六类表述、fail 三类结局都有处理规则与未知值拒绝。control 重算保留租约、不花普通尝试预算；state 重算需读新版本并计冲突预算；blob_condemned 由原种类写入方重传、同逻辑包重提。没有实现真实预算或任务终态写入。

HLD 对 segment_frontier_mismatch 既有独立结局也有 conflict 原因写法，本轮二者动作均 re_lease；具体 envelope 第二批收敛，没有凭空确定另一套协议。receipt / rejected 的处理语义明确，但回执消息结构尚未定义。

ID 输入为四元组，域分隔 `april.output-id.v1` + NUL + UTF-8 紧凑 JSON 数组，只含原样字符串和安全整数。固定向量、重传与不同段 / kind / index、Unicode 不归一化测试通过。本轮不创建 ID，也不让既有对象修订重派新身份；SDK 哈希 / 输出格式接入后必须复用这份材料，不能自行改拼接顺序。

## 5. 验证与实际结果

初轮 107 项（正式 64 + 原探针 43）中 106 通过，1 项断言假设 Literal 的 enum 内联而失败。检查实际 schema 发现复用类型正确导出为 `$defs/FetchStatus`，修正断言路径，不修改模型以迎合测试。

补预算扩张、UTF-8、中断区间和复现后 110 项通过；再补全 submission 语义和 Unicode C1 边界后，正式专用套件 79/79 通过。最终审查新增 opaque cursor 原样保留和 file / 切片导入无网绑定两例，正式套件共 **81 项**；最后完整 **make check：478 passed / 39 deselected，11.04 秒**，收集 517 项。八条 import-linter、静态连接器检查、迁移链 1 revision、正式 29 + 探针 3 schema 均通过。11 个新代码 header / 函数说明 / AST 与 git diff --check 通过。

真实 HTTP 测试采用共享 loopback socket + uvicorn / httpx，未知 status + 控制载荷经 TCP 往返不丢；结构损坏返回固定错误码，不返回输入，不挂到生产 app。原 43 项工具链测试继续通过。没有公网采集、模型调用或本地数据库 / Kafka 故障注入，本轮依赖远端 infrastructure / isolation 验证既有真实集成。

复现：

```bash
rtk proxy make setup
rtk proxy make test-first-contracts
rtk proxy make schema-check
rtk proxy make check
# 仅模型变化后显式生成并审查，不能用它掩盖漂移：
rtk proxy make schema-write
```

CI 仍是 checks / infrastructure / isolation。最终提交必须三项全成功再合并；实际 PR 信息交付时补记，不把先前提交的绿灯用于新 head。

## 6. 后续接手与回退

先读本 memory / ADR / first-batch-v1，再看 common / execution / fetch 与测试，之后看 transport 模型、SPI、SDK 类型别名和 registry / schema gate。第一批模型唯一来源已落地，勿从 probes 复制同名业务类型，勿把测试 codec 路由加入生产。

下一项 M0-04：完整结果包、字段投影 / semantic_bundle_digest、签名材料、13 步提交的完整请求 / 结局、事务上下文与锁约定、控制结构 / 权限、控制对象行与状态服务。把独立控制校验接到本批 raw 控制 JSON，未知页面枚举仍必须保留其他已合法控制；不要先 strict FetchPage 后受限投影。

M0-05 / 06 接运行时、SDK、文件连接器、证据、真实 I4 / 注册 / 撤回时补身份认证、流式预算、PG 权威和幂等、网络策略、持久连续前沿、动作 / 违规纯函数与全组合扫描。不把本次 81 项当作这些业务验收。G 模块的真实来源许可 / freshness / 有界投递仍开放，collection 仍禁用。

回退无数据库 / 业务数据迁移：撤销新增模型 / 协议 / 绑定 / 正式快照及入口，schema gate 恢复旧探针调用即可。实际消费者出现后必须按支持期 / 新版本演进，不能原地修改字段语义或 ID 输入材料。

## 7. PR 与交付

待创建本模块 PR。上一模块 PR #8 的实际 merge / CI 证据已补原 G memory；本轮不直接提交或推送 main。
