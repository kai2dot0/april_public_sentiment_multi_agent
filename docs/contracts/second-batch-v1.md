# 第二批契约 v1：提交、控制与同事务边界

本规范是 M0-04 的工程基线，依据总册 §3、§7.4 / §7.4.1、§8.2，大纲 §4.2 / §4.3 / §4.7 / §4.11 / §4.12，以及分册一 §1 / §3 / §4、分册二 §2 / §5 / §6。模型唯一来源是 `contracts/`；41 个公开根模型导出在 `contracts/schemas/v1-second/`。第一批 29 个模型和原探针的导出路径保持不变。组织角色签字没有被模拟。

## 1. 公共约定与接口分层

沿用第一批严格 JSON、安全整数、原样 UTC / 标识符、拒绝额外属性、字段 `digest/default_semantics/routable` 元数据。新增字段默认参与摘要；缺省值展开后才计算。嵌套 list / dict 仍可变，摘要 / 签名入口通过 `validated_snapshot` 从原生值重新严格校验，避免 JSON 序列化把 NaN 变成 null 后误接受。

| 边界 | 定义 | 本轮行为 |
| --- | --- | --- |
| I1 | `contracts.worker`、`spi.submission.WorkerService` | lease / heartbeat / submit / fail 消息、显式结局；没有网络服务或 Worker 循环 |
| I5 / I7 / I9 / I10 / I21 | `contracts.transactions`、泛型 TransactionalAdapter | 请求、受限事务端口、根 / 来源锁断言和步骤守卫；没有业务存储适配器或 Fake |
| I8 | `contracts.withdrawals` | 日志、注册、连续前沿、退役与待核验副本；没有注册服务 |
| I14 / I15 | `contracts.quality`、`contracts.normalization` | 六维记录结构与纯规范化 SPI；没有规则执行器或插件 |
| I17 / I19 | `contracts.availability`、`contracts.state_service` | 当前可用性 / 状态读取及版本结构；没有读取服务 |

`contracts` 与 `spi` 仍为互不依赖的叶子；Worker 侧具体绑定在 `sdk/worker_types.py`，平台事务绑定在 `runtime/transaction_types.py`。SDK 只导出共享摘要算法，不导出运行时 signer、PG 连接或提交权。

## 2. 完整结果包与载荷边界

ResultBundle 显式包括 Claim、ValidationRecord、state_writes、outbound_events、derived_writes、staged_promotions、evidence_creations、documents、PageResult、checkpoint_advance、normalization_tasks、continuation_tasks、coverage_declarations、pruning_records、按顺序排列的 committed_segment_digests。publication_policy 和 schema_version 也参与摘要。

Claim / 事件 / 标准文档使用 LogicalWrite：稳定引用 `(object_id, version, content_digest)`、钉定 `payload_schema`、JSON 载荷、根来源；派生写有 store / key / 版本化载荷 / 根。此 envelope **不等于 Claim、CloudEvents、文档载荷 schema 已定义或验证**。后续计算段必须加载钉定 schema，重算内容摘要，验证引用、槽位、引文、写入方和完整血缘。当前服务未开放这些输入。

collector 必须有 PageResult，其他 kind 不得携带页面控制、检查点或采集后继任务；collector 不创建规范化文档、分析 Claim、状态、派生或首次来源内容。EvidenceCreation 只允许 source_content / statistic / rule_decision；已有模型证据通过 staged_promotions 转正。任何语义对象都不携带 blob instance 改绑。

PageResult 包含完整开放 FetchPageWire、两张表版本、valid / violation、全部违规编号、最小 primary 和动作行。结构上 valid 恰有一个 1–18 动作，violation 不执行动作。这里只检查形状以及材料准备时的 V1 / V19 一致性；不是完整 19 条分类器。纯删除采集页仍完成控制与责任结算，不能标成 skipped_withdrawn；非采集包的普通输出全部确定性剔除时才提议 skipped_withdrawn。`proposed_terminal_status` 只做纯映射，不写任务状态。

## 3. 语义投影与摘要

唯一入口 `sdk.semantic_bundle.semantic_bundle_digest(SubmissionRequest)` 与平台共享 `contracts.semantic`、`contracts.canonical_json`：

1. 精确接受 SubmissionRequest，重新校验完整快照；不接受额外字段或自定义子类。
2. 展开所有默认值，按模型字段元数据递归投影。固定排除白名单只有路径 `lease_token`。未知排除声明会失败。
3. 动态 JSON 的所有键保留，内部名为 lease_token / receipt_id 的事实仍参与摘要；数组保留顺序，根集合要求调用方预先排序去重。
4. 对投影做 RFC 8785 JCS，输出 `sha256:` + 64 位小写十六进制。

大纲允许排除的 DB 代理键、回执 ID、非语义创建时间、心跳、传输凭据不应进入语义包。目前只有租约 token 实际存在于提交请求；其他类别未被引入，不使用全局键名黑名单。身份、修订、内容摘要、事件 ID、上游事件时间、来源位点、全部控制、剔除与早期分段摘要始终保留。

原 `runtime.core.canonical_json` 兼容重导出同一实现，原 RFC 测试继续执行；SDK 无需违反导入边界。JCS 依赖与版本不变，未增加依赖。

## 4. 签名材料与可信入口

ValidationMaterial 绑定 TaskBinding 的 tenant / caller / mode / namespace / policy_version / run_id / 发布策略 / 权限与配额、task_id、lease_generation、key_epoch、configuration_lock、input_snapshot；另含 operation、segment_no、after_segment、semantic_bundle_digest、verified_roots、pre_control_epochs、逐项 control_report、预选 source_locks、pruning_records。

`prepare_material` 要求调用方提供**平台独立解析**的根、永久代次、来源授权和来源配置。它核对任务、声明根与解析根相等、根可见性、状态写空间、manual_only 不降级、逐项控制权限、V1 / V19 和来源配置一致性。永久代次列表恰好覆盖全部解析根和已接受控制目标、排序且无重复；source 不属于永久代次。它没有实现证据查询、引用 / Claim 内容验证或 V2–V18，禁止把它直接当公开计算段。

RuntimeSigner 使用 HMAC-SHA256：签名字节为 `april.validation.v1` + NUL + JCS(`algorithm`, `key_id`, `material`)。verify 用 compare_digest，核对独立期望 TaskBinding、任务 / 操作 / 前沿和重新计算的请求摘要；真实租约有效期、作业 fencing、当前永久代次还必须锁内核对。签名不是来源 / 服务身份授权。

密钥仅可信运行时持有。Demo `from_file` 拒绝符号链接、非普通文件、非当前用户所有及 group / other 权限，读取 32–4096 bytes。真实部署须用安全随机源生成至少 32 bytes，不能把长度当随机性证明；文件不得入库 / 日志 / SDK。测试仅使用公开合成字节，不创建真实密钥。当前没有网络签名端点；轮换、key_epoch 持久化、旧租约绑定查询待运行时实现。

## 5. 分段、回执与提交优先级

- SubmissionRequest 要求 `segment_no == after_segment + 1`；complete 的 committed_segment_digests 长度恰为 after_segment，数组覆盖所有早期 checkpoint 段且顺序不改。当前前沿是否匹配由数据库权威判断。
- complete 回执键 `(task_id, complete)`；checkpoint 回执键 `(task_id, checkpoint, segment_no)`。操作不能互相复用回执。Demo Worker checkpoint 明确 unsupported，没有执行成功桩。
- 旧作业代次受限控制投影使用独立操作类别 `(task_id, control_projection, lease_generation, digest)`；同任务不同合法旧包分别幂等，只登记通过校验的控制，返回 conflict(lease_lost)，不变终态 / 键占用、不触发当前周期结算。
- conflict 五种原因沿用第一批处理表；control_epoch_changed / state_version_changed 保留租约并返回新前提，前者不花普通失败预算，后者计独立状态冲突预算。segment_frontier_mismatch 统一放进 conflict envelope，SDK 动作为重新领取。checkpoint CAS 诊断使用 lease_lost + checkpoint_version_changed。
- retry_after 原因只有 pg_admission / blob_condemned。后者由原写入方在事务外重传，再提交同一逻辑包；不能在第八步改绑 blob instance。权限 / 完整性拒绝与页面违规结局分开。

`COMMIT_STEPS` 固定如下；本轮只实现步骤 / 锁约束，不实现数据库业务编排：

| 步 | 顺序与待实现行为 |
| --- | --- |
| 1–2 | 任务行 → 有序消费者键占用行 |
| 3 | 先查普通回执 / 终态；代次变更终态的受限投影优先于普通摘要冲突，沿同一事务校验并登记控制 |
| 4–5 | 复核租约 / 作业代次 / 前沿；验签与任务绑定。旧代次分支也必须验签，不漏控制 |
| 6 | 文档 / 账号控制 → 来源 → 交接账号快照 → 撤回计数器；锁内比较永久代次、执行控制转换 |
| 7–8 | Manifest / 副作用检查；blob → 逻辑对象、证据与血缘 |
| 9–10 | 核心状态 → I9 观测 → I21 定时器；随后 I10 派生写 |
| 11–13 | 序号 / outbox → 最终检查点与 job CAS → 终态 / 释放；真实提交成功后才能对外回执 |

## 6. 同事务接口与守卫

PgSubmissionTransaction 只借用已有 INTRANS 的 psycopg 连接，记录 backend_pid / txid；没有 connect / begin / commit / rollback / cursor API。execute 只接受参数 tuple 和单条 prepared SELECT / INSERT / UPDATE / DELETE / WITH，返回值对象不暴露 cursor；结果行数有界，驱动错误仅保留固定消息和 SQLSTATE。原事务结束 / 换事务后借用失效。它服务可信 SQL 代码，不能防任意函数 SQL 或恶意 Python。

GuardedSubmissionContext 逐步推进 13 步，adapter 调用必须在对应步骤。每次调用重置根断言：I5 / I9 / I10 / I21 必须先 assert_roots，来源级记录改用 assert_source_locked，要求来源 NO KEY UPDATE。acquire_lock 先检查统一锁序再执行借用事务的锁 SQL，失败使上下文失效；SQL 实际目标 / 模式和登记键的对应仍由可信实现方保证，不能只登记不加锁。控制锁由 control_lock_plan 提前合并最强模式：引用 KEY SHARE、临时变更 NO KEY UPDATE、永久或合并变更 UPDATE；来源排序在文档 / 账号之后，不能晚升级。

异常使上下文失效；owner 必须整体回滚。退出 adapter 后保留的端口不能继续使用。static gate 检查 `adapters/transactional/*.py` 的常见网络 / 新连接 / 事务控制调用，当前 **0 个业务实现文件**，不把零文件通过说成实现验证。单元 RecordingPort 只记录守卫调用，不是业务适配器 Fake。真实 PG 实验验证事务 identity、提交前不可见、回滚无残留、多语句 / 提交拒绝、旧事务失效、错误脱敏。

## 7. 控制对象、保持与存储约束

控制身份为 `(scope, tenant, namespace, source, object)`。普通写根仅 doc / account，按 key 排序去重。tombstone / account_status / account_sync 逐项结构和权限校验；未知页面枚举不阻止其他合法控制被保留。失败项只留数组位置和固定原因，不留可执行信号或回显原文。只有 live collector、已认证 connector 上下文、同租户 / 空间且已授权来源可接受。

永久撤回仅 doc / account；来源 source_gap 是临时读门控，不能拒绝普通写或充当永久写屏障。大纲 §4.3 泛称 source 的旧表述与 §4.12 存在歧义，按后者及总册 §7.4.1 的明确锁语义处理，见 C04-06；若未来需要来源永久语义必须版本化，不能静默加入。

| DTO / 后续表 | 核心字段和数据库要求 |
| --- | --- |
| ControlObjectRow | 主键完整控制身份；permanently_withdrawn / withdrawal_epoch 与 availability_version 分离；各原因 enabled / epoch / comparison_basis / 最近位置与上游时间 / 事件身份；清除后仍保留比较键 |
| TemporaryReasonRow | account_inactive / sync_unverified / source_gap 分离，账号事件不得清 sync；resolved_cover_at 与覆盖证明独立于事件键 |
| HoldRecord | 账号 / 原因、hold_id、conservative / handoff_check、冲突类型与每次 merged_at、最后并入时刻、handoff_epoch、回访状态 / 尝试预算 / 关闭依据；后续建“同账号 / 原因未关闭”部分唯一索引及“同 hold 至多一个在途回访”约束 |
| HandoffRecord | 来源 / handoff_epoch、旧域与旧端可 null、新起点、建立 job / assignment / 时刻；来源控制行与建立代次关联 |
| TakeoverGapRecord | 来源 / gap_epoch 可多条空档；左端可 null、右端 / 依据 / 登记时刻，不覆盖旧空档 |
| HandoffAccountSnapshot | 来源 / handoff_epoch / account 唯一、adopted_at；I7 集合式生成，逐账号承接 |

来源配置固定 comparison_basis，不能逐事件或逐对切换；authority_position 账号信号需来源 SHARE，切换 / 接任报告需 NO KEY UPDATE。DTO 不执行事件排序、保守保持、回访覆盖证明、fencing、source gap 核验或 I16 控制转换；这些需要后续真实事务与并发夹具。

## 8. 状态、撤回、可用性与纯插件

I19 StateReadResult 区分 found / missing / forbidden / unsupported，found 必有正版本，null 可以是有效值；非 found 不携内容或血缘。状态写只能经 ResultBundle 在第九步 CAS，expected_version=0 代表不存在；上下文限定租户和写空间，不能由 Worker 直接更新数据库。

I8 全局序号用 uint64 十进制字符串，日志批次从 after+1 连续，不允许最大收到号掩盖缺口。广播只唤醒，PG 日志权威。注册需同进程组缓存清理证据，成功前不能 lease；前沿只在缓存实际失效后报告。退役必须已终止且已清空的可信证据，无法核对的前任副本保留 pending_verification。DTO 的布尔 / 引用不代替 OS 证据验证或 PG 持久确认。

I17 每项返回 readable 和明确阻断原因，不含正文，不支持不能当可读。实际服务须解析引用血缘、查询当前门控 / 文档核验 / 待确认撤回，不能按历史快照放行。

I15 normalize 只收 RawItem / 版本化 Meta，不得用 transport / DB，输出文档、source_content、observed 关系、互动和 exited_query。不能产墓碑，exited_query 不追溯历史统计。

ValidationRecord 绑定 Claim content_digest、规则包和维度目录的稳定版本；恰有 6 个唯一维度且含 HLD 明确命名的 computation，只有该维可有 affected_by_deletion。其余五维名称和规则在 HLD 未列明，本轮用钉定 dimension_catalog 表达，不发明产品语义；目录加载和成员检查尚未实现，测试 fixture-a…e 只是合成测试名称。异步 pending / 不适用不冒充通过。

## 9. 验证、版本与后续

运行 `make test-second-contracts`、`make check`，PG 使用显式专用 TEST_DATABASE_URL 执行 `make test-integration`。Schema 修改须 `make schema-write` 后审 diff，CI 只比对不修复。字段 / digest / 回执键 / 签名域变更属于跨组三级接口影响，应新版本和兼容测试；本轮没有生产消费者、业务表或迁移，既有第一批快照不变。

下一项 M0-05 先补齐首个文件闭环实际需要的 Claim / Evidence / 标准文档 / CloudEvents / Manifest 等钉定 schema 和完整动作 / 违规判定，再做最小运行时、SDK、文件连接器、证据 API、I5 / I7 / I21 真实适配器与明确注册 Fake。M0-06 替换真实注册、补 I9 / 派生屏障、撤回 / 进程控制与故障矩阵。当前不是 M0 出口或 H11 已通过。
