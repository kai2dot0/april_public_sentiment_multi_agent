# M0-05B：首个采集闭环的载荷、配置锁与签前绑定

日期：2026-11-04（Asia/Shanghai）。基线 main `9d249312c7d25abc4419328e715478738d1c4be0`；功能分支 `feat/m0-collection-config`。遵循独立分支 → 本地验证 → PR → 实际 CI → merge；没有直接向 main 提交。

## 1. 背景、依据与范围

用户要求继续下一项开发，并持续保留可接手的 dev_memory。M0-05A 已完成正式采集判定器，但缺少把页面里的具体证据、事件和任务配置绑定到签名的实现。本轮 B 补齐首个文件闭环需要的契约与计算逻辑，后续 C 再落真实存储与适配器。

开发前阅读大纲 §4.1 Event、§4.3 Evidence / 标准文档、§4.4 Manifest、§4.6 Run / Task、§4.7 提交，以及已有 §4.11 / §4.12；对应分册一 §1 / §3、分册二来源 / SDK 设计、总册 §5 / §7.3 / §7.4.1 / §8.2–8.4。职责按总册最新调整；文档内组织评审要求没有被模拟为实际批准。

总册 §11.1 三级接口影响：新增 v1 模型与调用入口；没有旧业务消费者、SQL 迁移、新依赖、锁序改变或 A/B/C 裁剪。Claim 与标准文档具体载荷随 normalizer / Agent 链路实现，B 仅对文件一页 → 证据引用 / 事件 → 检查点的输入做可执行验证，不把整个 M0-05 标为完成。

## 2. 实施方案与核心文件

| 文件 | 职责与关键约束 |
| --- | --- |
| contracts/artifacts.py | 严格 SemVer / URI、统一模型与 Schema 内容引用、五项依赖 flags OR / 不清除检查 |
| contracts/evidence.py | 六种证据及写入角色表、来源 / 父引用 / 根、不可变逻辑身份、实际字节摘要、独立治理版本 |
| contracts/events.py | EventIntent / EventRecord / EventSchema、首个 CollectionPageData、逻辑事件写入构造 |
| sdk/events.py | 严格 JSON 与 CloudEvents 顶层标量转换、causedby、inline / reference、排序 / subjectseq 检查 |
| contracts/manifests.py | 全部 Manifest 声明、连接器时界 / 协议元数据、显式部署与池缺省、资源 / 租约 / Demo 限制 |
| contracts/collection_config.py | 来源资格快照、事件注册快照、完整有效配置、输入 / file query / task key / task definition |
| contracts/collection_registry.py | 独立注册 16 个公开根模型，并从真实模型生成 Schema 内容锁 |
| runtime/core/collection_config.py | 受信快照装配、缺省展开、权限收窄、来源门禁、配置冻结、任务身份与重建 |
| runtime/core/collection_payloads.py | 实际证据目录解析、独立根并集、事件 / coverage / 检查点 / PageResult 校验、签前绑定和外发记录构造 |
| contracts/identities.py；sdk/output_ids.py | 共享稳定输出 ID；SDK 仅重导出，避免平台与 Worker 分叉 |
| contracts/bundle.py | PageResult / coverage 稳定身份及 JCS 引用 helper；原模型字段不变 |
| contracts/tooling.py | 从已有严格线解码抽出 decode_json_object，CE 与 decode_wire 共用重复键 / 编码 / JSON 边界 |
| runtime/core/validation_signatures.py | 违规 collector 页忽略普通写入根，原请求仍完整参与包摘要；具体入口先平台重算判定 |
| tests/support/collection_config.py | 合成数据构建器，实际调用解析器 / ID / 引用函数，不提供业务 Fake 成功替身 |
| tests/unit/test_collection_{artifacts,config,payloads}.py | 三类专用正反例，重哈希伪造、来源 / flags / checkpoint / registry / withdraw 限制 |

新增 Python 文件带最顶部 Codex(GPT) / 实际本地生成日期 header；新增函数有说明。JSON 快照不加注释。`Makefile` 增加 `test-collection-config`；schema checker 增加 v1-collection。没有改 uv.lock、Compose 或 Actions 工作流。

## 3. 接口、身份与信任边界

先提供平台受信的 manifest / defaults / source / registry / context / plan / policy，再调用 resolve_collection_configuration，随后保存 configuration_reference。来源 enabled / qualification_ref / producer.role 是快照字段，不能替代认证、合规或平台注册。部署缺省不写死在代码，缺失必要池 / 权限就失败；只以 None 继承，0 和 [] 保留。资源单位、心跳 / 租约、max_batch=1、worker_checkpoint=false、可重建缓存和显式能力都检查；工具、池、外读权限只收窄。来源控制送达界与 late arrival 分开，原声明和覆盖都过 guarantee 门禁。

配置锁覆盖展开后完整模型、Run / source / manifest / registry / deployment 引用、正式判定版本与两个规则文件摘要、16 个实际 Schema 锁。verify_configuration 比对平台任务里的预期引用，不能允许 Worker 同时换内容和摘要。当前非空 domain_packs / knowledge_services / predicates 明确不支持，等 M1 实际解析器接入；空快照不是“加载成功空结果”。

prepare_collection_task 钉住 FileQuery 字节摘要与逻辑 file_id（无任意宿主路径）、输入引用、flags、发布限制、优先级和捕获检查点版本。周期任务身份包含 cycle，连续任务不随窗口切换变身份；两者都绑定 job / generation / slice_seq。实际创建 / 领取前仍须 PG 锁内检查 generation / fence。Fake 强制 contains_simulated_input，manual_only 不解除，Null 和受控导入明确拒绝。

EvidenceRef 摘要来自包含出处 / 制品 / 父链 / byte_digest / compliance / roots 的逻辑清单，URI 与 content_digest 必须一致且 version=1；同字节不同出处不共用逻辑身份。verify_evidence_bytes 才校验实际内容。治理单独版本化，不把分级 / 保留 / blob instance 混入逻辑身份。初始根真实性与父链解析仍依赖后续受信证据服务。

结果包只放 EventIntent；平台提交后再补 emittedat / traceparent / subjectseq / hops 到 EventRecord，避免普通投递元数据影响语义幂等。SDK 统一严格 JSON 解码，wire 明确展平 routing 与 causedby，去掉内部 content_kind；必需 CE 上下文不可由模型缺省掩盖，可选 null 视为未设。事件只允许对象 JSON，总信封 1 MiB，metadata 内联严格 <16 KiB，source 永远引用。subjectseq 是正 uint64 字符串，causal 才能出现。first-loop registry 必须完整等于内建 arrival 元数据描述，不能自报尚无映射的 attr 别名。通用 codec 能表达 causal 不等于服务已实现生产者唯一血统 / 排序计数器。

prepare_bound_collection_material 重建配置与任务，核对实际 TaskBinding，再平台完整重判 PageResult。valid 页从可信清单独立取根，核对 promotions / 返回 raw_refs / 事件全部引用、输出身份与 Schema、配置 / PageResult / coverage / query、flags / 发布策略和捕获检查点。引用式元数据须取回并重算摘要。任何遗漏 checkpoint / coverage、替换后重哈希的事件字段、租户 / 来源 / 制品变化都会拒绝。

violation 页普通写入由后续受限提交忽略，但原请求仍进入完整语义摘要；独立合法控制保留，不把被忽略写入的 Worker 根当实际根。内容根遇正永久 epoch 或同包合法墓碑时，当前拒绝普通内容并要求存储剔除；pruning_records、后继任务载荷不能假装验过。最终材料仍交已有 HMAC / transaction guard；本函数不持密钥、不签真实业务回执、不提交事务。

## 4. 实际验证

最终本地 `rtk proxy make check`：**884 passed / 48 deselected，18.44 秒，收集 932 项**。其中新增专用测试 **160 项（81 artifacts、43 config、36 payloads）** 全部被该门禁实际执行。八条 import-linter 边界、连接器静态边界、事务静态检查（仍 0 业务实现）、迁移链（1 revision）及五组 Schema **3 + 29 + 41 + 2 + 16** 均通过。正式根模型合计 88，连探针共 91；此前 29 / 41 / 2 正式快照未改。

测试覆盖严格 SemVer / URI、共享输出 ID、五项 flags、证据 kind × role 全组合、治理不改身份、实际字节、父引用、CE 编解码 / 大小 / uint64 / 畸形 JSON、缺省与权限、保证覆盖降级、内容锁 / 任务重建、Fake / Null、无实现别名、签前 payload 篡改与完整引用、违规控制保留和撤回内容拒绝。没有用这些单元测试声称实际服务集成成功。

本机没有重跑无变动的 PG / Kafka / cgroup 故障；最终 PR 必须实际通过 checks / infrastructure / isolation。checks 仍跑 M0-05A 全量 2,764,800 判定并上传 fetch-decision-evidence，实际远端结果以 PR 最终 head 记录为准。

14 个新增 Python 文件的 AI header / 日期与 93 个函数说明通过 AST 检查；9 份相关 Markdown 本地链接存在，git diff --check 通过。旧正式 Schema 三组 git diff 为空。

## 5. 后续接手与限制

1. 读本记录、[消费规范](../docs/contracts/collection-config-v1.md)、[ADR 0012](../docs/decisions/0012-collection-config.md)、[规则映射](../docs/rule-map/m0-collection-config.md) 及 M0-04 / A memory；运行 make test-collection-config 与 make check。
2. 下一项设计并落真实 PG 证据目录 / blobs / staged holders、任务 / 作业 / generation / checkpoint / outbox、控制 / epoch / 定时表；应用显式 psycopg SQL，SQLAlchemy 只供 Alembic。新迁移同步调整期望版本及真实专用库测试。
3. 实现证据 API 与 I5 / I7 / I21 同事务适配器；遵守已冻结 13 步提交与根 / 来源锁序，真正处理剔除、epoch 重读、CAS、回执幂等和原子写入。不能以 Fake 代替这些适配器。
4. 任务存储是 configuration_lock / TaskBinding / 原始输入的权威；证据服务是清单、初始根、父链、可用性与服务认证的权威。接入 B helper 时拒绝 Worker 自报的“受信快照”。
5. 接最小运行时 / SDK / file 读取与明确 I8 注册 Fake；必须先确认注册再 lease。做到文件一页 → 提交 → 检查点 → 事件，才能声明首个集成点完成。I9 不在该最小集成点调用；M0-06 接真实注册 / 撤回 / I9 与故障矩阵。
6. Claim / 标准文档在 normalizer / Agent 链补具体模型、唯一写入方与签前校验。模型网关 / 真实来源 / 分析 / 控制台 / 报告 / 受控导入仍按 M1–M3，当前没有可供用户操作的业务页面。

回退移除新增模块 / Schema / 测试 / make target、恢复基础 helper 调用；本轮无业务数据或迁移可回滚。真实消费者出现后身份算法 / Schema / 发布策略与路由 profile 变化需版本化，不能静默重释。

## 6. PR 交付记录

PR 尚待创建；此处将在同一功能分支补入实际链接。最终受检 head、三个实际 CI job、完整判定附件及 merge SHA 记录到 PR 描述，避免为自引用 SHA 反复改动受检内容；下一模块可在本文件补记最终合并事实。已有授权允许 PR merge，不直接提交 main。

上一项 PR #11 实际最终 head / CI / artifact / merge 证据已补到原 A memory。GitHub 连接器此前创建实际返回 integration 403，因此写入使用已有授权 kai2dot0 的 gh 后备，保留全局默认账号，不输出 / 落盘令牌；这不是自动审批拒绝，也无需重新申请 workflow 权限。
