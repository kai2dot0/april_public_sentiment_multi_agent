# 第一批契约 v1：调用方规范

对应 M0-03、HLD 总册 §3 / §8.2 的第一批。执行上下文和 FetchPage 的 `schema_version` 为 `1.0.0`，整个 ABI 由 `contracts/schemas/v1/` 管理。文件是工程实施规范，不代替文档要求的组织签字，也不宣称完整 M0 验收。

## 入口、字段政策与数值

从 `contracts.execution`、`contracts.fetch`、`contracts.transport`、`contracts.states`、`contracts.identities` 导入模型；注册表在 `contracts.registry.CONTRACT_MODELS`。未知字段禁止，基本类型严格，缺省逐字段记录；普通 dump 展开默认值。所有本批字段摘要政策为 include。后续语义包投影不能让插件自行排除字段；签名与结果包在 M0-04 实现。

不可信 HTTP / Worker JSON 必须用 `decode_wire(Model, bytes)`，固定异常 `WireDecodeError` 不回显输入；不要直接用宽松类型转换、`model_construct` 或默认的 `model_validate_json` 接受不可信数据。直接 Python 构造使用同一模型和严格类型。没有生产路由在本轮被挂载。

计数 / 代次范围 `0..2^53-1`，需要已生成的段号和指定版本时从 1 起；外部 seq / 时间游标 / opaque cursor 的 value 使用字符串。动态 JSON 的大整数也必须编码为字符串。UTC 时刻使用合法的 `YYYY-MM-DDTHH:mm:ss[.1至6位小数]Z` 文本；不重写精度，不做时区转换。标识符非空、无空白 / 控制字符 / 非法代理项，不 trim 或 Unicode 归一化。

## 执行上下文与模式

ExecutionContext 必填租户、调用主体、策略版本、run、模式、写空间、可见空间、能力 / 工具 / 池、显式配额、发布策略、外部读取授权与外部副作用开关。能力 / 工具 / 池不得重复，每个允许池恰好一个配额；空能力 / 外部授权表示无权限，缺字段不表示自动允许。

| 模式 / 来源 | 写空间 | 外部采集 / 读取 | 外部副作用 |
| --- | --- | --- | --- |
| live | formal | 已显式授予的按作业连接器读取 | 仍需 Manifest 与平台授权；上下文开关不自行授予 |
| replay / shadow / eval | isolated | 禁止重新采集；其他读取仅显式 canary | 禁止 |
| backfill | formal | 内部补算，无外部读取授权，不运行 collector | 禁止 |
| slice_import | live + formal | file 连接器，无外部读取；保留 source_namespace | 禁止；publication_policy=manual_only |
| slice_reproduction | replay + isolated | 读既有切片，不重新采集；保留 source_namespace | 禁止 |

发布策略的 v1 名称为 `automatic` / `manual_only`：automatic 仅表示受平台策略判定的自动发布候选，并不跳过报告就绪或审批条件。合并时取更严格政策的业务实现尚未接入。上下文不可从 JSON 的 caller.role 或 frozen 属性获得真实权限，平台入口必须重建 / 核对认证上下文。

`validate_collection_plan` 是 Demo 计划准入纯函数：collector 仅 live；long_session 拒绝；continuous bounded_slice 必须支持恢复；continuity=required 必须有相应有效能力声明；history 必须 manual_only；control_delivery_bound=guarantee 只允许 http-fixture + test。能力必须来自平台合并后的有效配置，不能把连接器自报值当已授权。

## FetchPage 的两个入口

FetchPageWire 包含 HLD §4.11 的全部页字段：items、item_outcomes、checkpoint_before / after、status / status_reason / failure_scope / end_reason / completeness、retry_at、metering、coverage、continuations、tombstones、control_events、suggested_action。

- 必填 status、completeness、coverage 和两个 checkpoint 字段；checkpoint 值可以显式 null。集合缺省为空，metering 缺省为零计数，其他可选值缺省 null。
- items 是 `{item_ref, raw_ref, observation?}`；raw_ref 是逻辑对象 `{object_id, version, content_digest}`，不带 blob 实例 ID。item_ref 用于对应条目寻址与观测集合。
- item_outcomes 是 `{item_ref, outcome, reason?, observation?}`；missing 不直接生成墓碑，failed 不充当删除证据，present 不直接确认可读。观测可携带源快照下界、存在上界、依据引用和 source / account 门控代次，均不由 observed_at 自动推断。
- 线模型所有页面枚举保留原始字符串，包括未知、空串和带空白值。status_reason 仅 forbidden 下按枚举识别，其余为自由诊断。`unknown_fetch_enum_paths` 只返回 V1 的未知位置，不返回完整 verdict。
- FetchPage 构造侧使用已知枚举，forbidden 要有已知原因；它不代替 V2–V19 或动作表。线模型不能先转成此模型再判定，否则墓碑会在未知值时被丢弃。
- tombstones / control_events 是本批明确保留的 JSON 对象数组；第二批独立控制契约决定其中各项是否合法 / 有权执行。本轮不登记控制、不做受限投影、不据 JSON 中时间建立可信接收证明。
- suggested_action 只是建议，运行时不能据此调度。完整 18 行 / 19 条 YAML 与同一纯函数判定器在运行时 / 决策表接入阶段交付，当前没有 H11 通过声明。

FetchDecisionContext 由平台构造：调度、bounded_slice、单资源标记、请求方式、请求集合、已持久前缀集合、有效前缀标记及控制校验结果。单资源必须恰好寻址一个条目；“单资源 × 查询或流”不能构造，集合身份不能重复。Worker 自报 control_validation_passed 不具权威。

## 位置域与交接

SourcePosition = domain + 非空字符串 value。domain 明示来源、上游实例、排序域和游标语义版本。value 是 opaque cursor，保留原始空白 / UTF-8 文本，不套用标识符去空白约束；具体合法性由钉定来源协议解释。不能按字符串或数字大小跨域比较。Checkpoint 另存连接器 schema / opaque state、位置、权威指定版本与 continuous_receipt_start；起点未知保留 null，不能用第一条事件补造。

Coverage 可带位置域、当前位置、switch_report、takeover_report 和来源特定事实详情。切换报告记录旧域 / 旧最后位置（可未知）、新起点、钉定映射及其验证标记；接任报告记录指定版本 / 连续接收起点。这些是待验报告，不自动建立 handoff 或授权。报告需在正常提交前反复携带，正常提交即使不推进位点也应记录相应元数据；实际持久化在后续运行时实现。同包两报告的处理先后按 HLD，尚无控制状态执行。

## I4：SDK 句柄与传输服务

`spi.transport` 只声明泛型 Protocol，不依赖 contracts；`sdk.transport_types` 绑定正式模型，保持现有独立叶子边界。Protocol 不能直接实例化，不提供成功空结果或生产 Fake。

| SDK 绑定类型 | 操作 | 约束 |
| --- | --- | --- |
| TransportHandle | request / open_websocket | 绑定上下文、凭据范围引用、io 配额与切片预算；不暴露底层客户端 |
| WebSocketHandle | receive / send / record_receipt / confirm_caught_up / close | 帧与发送消息按 Base64 保留字节；text 必须 UTF-8；公开 record_receipt 只接受 frame_ref + upstream_event_identity |
| AdmissionService | admit / report_throttle | 每次请求先经 PG 权威准入，许可事务提交后再联网；持久冷却；Redis 仅拒绝镜像 |
| RealtimeRegistrationService | register_connection / record_receipt / report_interruption | 先登记后订阅；可信 SDK 才附接收时间 / 帧序号 / 追赶标记；失败不阻塞已运行连接 |
| AuthorityAssignmentService | assign_authority | 仅已认证运维身份；来源级唯一权威作业，每次生成新指定版本，保留操作人与依据 |

TransportBinding 的非空 grant 必须已在上下文中；file / 切片导入可显式 grant=null，此时 origins / methods 必须都为空，句柄没有任何网络访问资格，真实实现须拒绝联网操作。预算不得超出 io 的请求 / 字节 / 墙钟配额；消息和页数额度仍来自显式切片预算。origin 是精确源站，不是 URL 前缀。URL 结构检查拒绝 userinfo、fragment、错误 scheme / 端口、控制字符；不替代代理白名单、DNS / TLS / 重定向执法。连接器请求不得覆盖 Authorization / Cookie / Host / Upgrade 等保留头。真实凭据不出现在模型中。

HTTP 响应和 WS 字节统一 `payload_b64`，使用唯一标准 Base64；null 请求 body 与空 body 不等同。头用有序数组保留重复项；拒绝 CR / LF / NUL。网络读写、流式内存上限、共享重试预算、认证、许可消耗与持久化实现仍在后续阶段。

recorded 接收结果必须含 durable receipt_ref 和同一最早记录的 first_received_at / first_catching_up；放弃登记返回 unregistered 且不得携带时间证明。按（connection_id，frame_seq）幂等，事件首次值取所有记录最早记录；不能用确认 / 再导入时间替换。中断审计的帧号与时刻不得反向。catch-up 解除需要来源协议可验证的依据；合同里有 proof_ref 不表示依据已通过验证。

## 状态 / 错误语义

页面的 11 个 status、completeness、end_reason、failure_scope、forbidden 原因、条目 outcome 均在 `contracts.states` 唯一声明。任务八态和作业四态同 HLD。已有技术探针保持各自历史模型，不对探针原型做破坏性改义。

| 原因 | SDK 处理语义 |
| --- | --- |
| digest_mismatch | 告警，不重试 |
| lease_lost | 放弃当前任务 |
| control_epoch_changed | 保留租约，以新控制前提重算；不消耗普通尝试预算 |
| state_version_changed | 重读状态并重算，计入冲突重算预算 |
| segment_frontier_mismatch | 重新 lease |
| pg_admission | 对同一请求退避重提 |
| blob_condemned | 由该证据种类写入方重传，保持同一逻辑结果包后重提 |

尚未形成 FetchPage 的执行失败使用 WorkerFailure：输入撤回 → skipped_withdrawn；可重试 → 平台持久预算重试；否则死信。错误只传 error_class / sanitized diagnostic_code，不传异常原文。未知冲突 / retry 原因明确失败，不能盲目重试。

HLD 有将 segment_frontier_mismatch 单列返回和列在 conflict 原因中的两种表述；第一批只冻结其“重新 lease”语义，具体响应 envelope 在 M0-04 明确。receipt、rejected 及完整提交结局结构也在第二批，本轮不提前生产回执。

## ID 派生输入

新输出使用四元组 `(task_id, segment_no, output_kind, local_output_index)`，task / kind 是原样标识符，segment 从 1 起，local index 每种输出类型从 0 起。complete 的最终段为 after_segment + 1，不复用最后 checkpoint 段。已存在引用和对象修订不得调用新 ID 分配规则。

`id_derivation_material` 返回：ASCII 域分隔 `april.output-id.v1` + NUL，再拼 UTF-8 紧凑 JSON 数组 `[task_id, segment_no, output_kind, local_output_index]`。字符串不归一化、不转义为 ASCII；整数都在安全范围；不拼接裸字符串，不依赖 dict 顺序、Python hash、平台字节序或 UUID 随机数。本批只冻结输入材料，SDK 统一哈希 / ID 输出函数随 SDK 接入，不提前创建业务对象 ID。

固定向量：`("task",1,"event",0)` → `april.output-id.v1\0["task",1,"event",0]`（`\0` 代表单个零字节）。同段重传材料一致，task / segment / kind / index 任一变化材料不同。未来修改格式必须新增版本和兼容说明，不能原地改义。
