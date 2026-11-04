# 首个采集闭环：载荷与配置锁 v1

M0-05B 的可执行契约与纯计算入口。依据大纲 §4.1、§4.3、§4.4、§4.6、§4.7，以及总册 §5、§7.3、§8.2–8.4。没有任务服务、证据 HTTP 服务或文件读取器；不能用本页的模型校验结果代替平台注册、授权、数据库提交或投递确认。

## 配置装配与冻结

`runtime.core.collection_config.resolve_collection_configuration` 接收平台取得的 Manifest、部署缺省、来源资格配置、事件注册表、执行上下文、采集计划及正式判定策略。`as_of` 必须显式提供，函数不查询时钟、网络或密钥。`source.enabled`、资格引用、角色和制品字段只是受信快照的表达，接入服务必须先认证并校验这些快照的来源。

Manifest 包含插件版本 / 制品 / owner / impl、订阅与输出、状态 / 比较 / 迁移、缓存 / 合并 / 计时器、资源 / 租约 / 心跳 / 重试 / 死信 / 优先级 / 丢弃 / 配额、工具 / 副作用、领域包 / 知识 / predicates、验收 / 评测 / flags / resume / 能力及连接器声明。SemVer 按 2.0.0 校验。资源单位固定为 cpu millicores、memory bytes、wall_timeout milliseconds；租约、心跳、TTL 和时界也用 milliseconds。

缺省必须来自有版本的 `DeploymentDefaults` / `PoolDefaults`；只以 `None` 触发继承，显式 `0` 优先级和空重试列表保留。展开后检查资源上限、授权池与工具、心跳小于租约、Demo max_batch=1、worker_checkpoint=false、死信开启及可重建缓存；执行上下文只收窄权限。`manual_only` 扇入不解除，优先级取继承值的最大值，依赖 flags 逐项 OR；Fake 强制 simulated，Null 不创建可执行任务。

来源覆盖与原声明都经过采集计划门禁。`late_arrival_bound` 与 `control_delivery_bound` 独立；只有 http-fixture 的 test 环境可以使用控制送达 guarantee，不能用覆盖为 unknown 隐藏非法原声明。真实来源仍沿用协议核实登记表的 unknown 与禁用状态。file 声明必须无网络 allowlist、凭据与外部副作用。

`configuration_reference` 对完整展开模型做 JCS / SHA-256；锁包含 Manifest / 部署缺省 / 来源 / 注册表引用、完整有效值、Run 上下文、决策策略文件锁与当前 16 个根 Schema 的实际内容锁。没有领域包 / 知识 / predicates 的解析器时明确拒绝这些依赖，空锁仅用于无此依赖的首闭环。`verify_configuration` 要求与平台任务记录里的 expected 引用一致；Worker 自行更换摘要不构成授权。

## 输入与任务身份

`FileQuery` 仅含受控 file_id、实际文件字节摘要、jsonl 格式和页大小，不接受宿主路径或任意 URL。`InputSnapshot` 排序去重稳定输入引用并捕获 flags、发布策略、优先级、检查点及其版本；query 也进入输入锁。

`prepare_collection_task` 生成不可变 `CollectionTaskDefinition`。任务 ID 使用独立 v1 域与 JCS：periodic 包含 job / generation / slice_seq / cycle；continuous 不把窗口 cycle 纳入身份。`verify_collection_task` 逐字段重建，不能只核对 task_id。当前仅原生 file bounded_slice 计划；受控切片导入涉及保管库与前沿预置，明确 unsupported，留到 M3。任务创建、generation/fence 重查、lease、预算及作业锁都由后续 PG 服务实现。

## 证据身份

`EvidenceManifest` 区分 raw_response、source_content、model_output、statistic、rule_decision、agent_intermediate，校验大纲的唯一写入方角色表。角色字段不授予实际创建权限。raw_response / source_content 必须有来源、标准文档身份与修订；父证据是排序去重的真实 v1 EvidenceRef，根必须与租户 / 来源一致。

`evidence_reference` 对不可变逻辑清单做 JCS / SHA-256，返回 `evd://sha256:<hex>`、version="1" 与相同 content_digest。清单包括生产服务 / 制品、父引用、字节摘要与合规引用；同字节不同来源不共享逻辑身份。`verify_evidence_bytes` 独立计算实际字节摘要。blob instance 和可变治理不参与逻辑身份；`EvidenceGovernance` 以该证据引用与独立版本记录分级、保留、访问范围及审计依据。

当前没有上传 / holder / blob / GC / 访问授权与撤回服务。调用者须从可信证据服务提供清单，不能把 Worker 自报目录作为已解析证据。根的初次真实性及父链解析仍由该服务承担。

## 事件语义与线格式

`EventIntent` 是结果包内的语义意图；`EventRecord` 是平台补充 emittedat / traceparent / subjectseq / hops 后的记录。外发时间、追踪和平台分配序号不进入普通意图，重试不会因它们改变包语义。`finalize_event` 只构造记录，不能证明序号已分配或消息已投递。

`sdk.events.encode_cloud_event` / `decode_cloud_event` 显式转换 CloudEvents 1.0 JSON：`routing` 中已登记的 attr* 别名展平为顶层标量，causedby 列表转逗号分隔字符串，内部 content_kind 不外发。EventRecord Schema 是内部记录格式，不能直接作为 CloudEvents 线 Schema。接收端严格拒绝重复键、非法 JSON、缺少必需 CE 上下文及未知存储字段；可选上下文 null 视为未设置。上下文整数限 int32，subjectseq 是正 uint64 十进制字符串且只用于 causal；causedby 最多 32 个无逗号 ID。

本项目 profile 只允许 JSON 对象载荷、application/json，总信封最多 1 MiB。元数据内联必须小于 16 KiB；来源内容一律 dataref + datadigest，不能因尺寸小而内联。URI 只做无网络的语法边界检查，不表示对应资源可访问或已授权。

首闭环唯一可装配事件为 `april.collection.page-recorded.v1`，数据 Schema URI 为 `urn:april:schema:collection-page-recorded:1`，arrival 排序。`CollectionPageData` 只含任务 / 作业 / 来源 / 配置 / PageResult / coverage / raw_refs / flags。运行配置中的注册项必须与内建描述完整一致，拒绝未实现的别名映射。通用 codec 可表达其他描述，但不表示运行时已支持其业务处理。订阅只允许已登记路由字段的等值过滤；分区表达式 v1 只取一个路由属性名，不解释任意代码或 data 路径。

## 签名前具体绑定

正式文件入口使用 `prepare_bound_collection_material`，不能只调用早期基础签名 helper：

1. 对比受信 `TaskBinding.configuration_lock`，重建任务与输入锁，核对调用上下文、查询摘要、检查点及调度。
2. 重算 PageResult 稳定 ID / 摘要，再调用 M0-05A 完整动作 / 违规判定器和独立控制校验。
3. valid 页面解析 raw promotions 的不可变证据清单，核对实际引用、租户、来源命名空间、制品和合规身份，从清单独立取根并集。
4. 校验事件的本段输出 ID、逻辑写入摘要、具体载荷 Schema、上下文、发布限制、全部 raw 引用与 flags；引用式元数据必须取回并重算 JCS 摘要。
5. coverage 与检查点必须对应同一 PageResult / query / generation / cycle / 捕获版本；是否需要推进由判定动作独立决定。
6. 当前不执行永久撤回剔除；发现正 epoch 或同包合法墓碑与实际内容根相交时明确拒绝，要求后续存储解析器处理。pruning_records 与后继任务载荷同样不能假装验证成功。
7. violation 页面普通意图不进入实际根集合，仍纳入原始包摘要；只保留受限提交允许的控制。平台重算完整判定后才准备已有 HMAC 材料。

这仍是计算段，不持有密钥或提交事务。最终签名、epoch 锁内重查、根重建、实际剔除、13 步提交与持久化回执需后续服务接线；不能将本函数返回当作 commit 成功。

## 检查与后续

`make test-collection-config` 运行三份专用单元测试，`make schema-check` 检查五组快照，`make check` 包含全量本地门禁。16 个新增根模型在 `contracts/schemas/v1-collection/`；此前 72 个正式 Schema 不变。

下一项先落真实证据 / 任务 / 作业 / 控制表和 I5 / I7 / I21 同事务适配器，再接 SDK、文件 IO、最小运行时和明确的 I8 注册 Fake。Claim / 标准文档具体载荷随 normalizer 与 Agent 开发，M0-05B 没有把它们标为已实现。详见 [ADR 0012](../decisions/0012-collection-config.md)、[规则映射](../rule-map/m0-collection-config.md) 和 [接手记录](../../dev_memory/2026-11-04-m0-05b-collection-config.md)。
