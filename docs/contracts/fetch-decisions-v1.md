# 采集判定器 v1

依据大纲 §4.11 的 18 行动作 / 19 条违规、分册一 §1 / §3、总册 §8.3 / §8.4。M0-05A 实现正式判定与计算段接线，服务于后续文件闭环；尚无任务数据库、计时器执行或作业状态变更。

## 1. 权威输入与执行顺序

`contracts.fetch_decision_rules.decide_fetch(page, context, policy, expected_lock)` 接收正式开放 FetchPageWire、平台 FetchDecisionContext、已加载只读策略和钉定配置锁。页面与上下文先用已有 validated_snapshot 重验，防止嵌套可变值绕过；不会从 suggested_action 推导调度。

context 的请求方式、请求条目集合、可靠持久化前缀、是否单资源以及控制通过标志都必须来自平台。生产代码调用 `runtime.core.collection_decisions.classify_collection_page`：先从认证执行上下文和来源授权逐项重算控制，再覆盖 context 的控制标志，最后做完整判定。不能把 SDK 预跑结果作为权威。

1. 按编号执行所有 V1–V19，返回全部命中码，排序去重，最小编号为 primary。
2. 有任一违规则返回 violation，action_row=null、next_action=violation、job_target=paused、不推进检查点、不允许普通内容输出；**不执行动作表**。
3. valid 输入计算全部动作匹配数，必须恰好一行。零行或多行是策略错误，抛固定 DecisionPolicyError，不用第一行或默认 idle 掩盖。

未知枚举是 V1；forbidden 原因未知同时命中 V1 / V2，非 forbidden 原因是自由诊断文本。不同违规可同时成立，不因 V1 提前返回。非法控制只保留位置 / 固定原因；其他独立通过的墓碑 / 账号信号仍保留在 ControlValidationReport，供后续受限控制投影执行。

## 2. 观测集合与 V8

| 页面 / 请求 | 可接受的 item_outcomes |
| --- | --- |
| ok、按条目寻址 | 请求集合内的 present / missing / failed；单资源 ok+missing 另触发 V18 |
| ok、查询 / 列表 / 流 | 本页返回集合内的 present；missing / failed 及未知 outcome 均不满足“只允许 present” |
| 非 ok、有有效前缀 | 已可靠持久化前缀内的条目结果，仍独立检查枚举 |
| 非 ok、无有效前缀 | 不允许条目结果 |

是否有前缀不从 items 或 Worker metering 猜测。正式上下文拒绝“single_resource × query_or_stream”。规范没有增加第二十条违规；其他详细载荷 / 引用 / 重复事实校验由后续业务契约和计算段负责。

## 3. 版本与规则文件

- `contracts/decisions/fetch-actions.v1.yaml`：fetch-actions/1.0.0，必须恰好 18 个连续编号。
- `contracts/decisions/fetch-violations.v1.yaml`：fetch-violations/1.0.0，必须恰好 19 个连续编号，不能携动作 effect。
- `DecisionPolicyLock` 同时绑定两表的 version、原文件字节 SHA-256、evaluator_version=fetch-decisions/1。文件注释 / 空白变化也改变锁；不能仅凭同名版本接受变更。
- 两表显式登记当前正式 FETCH_ENUM_DOMAINS；枚举增加、删除或排序变化会使加载失败，必须更新规则、版本及全量证明。

项目采用 YAML 1.2 的 JSON 流式语法子集与首部注释，以现有严格 JSON 解码器加载，避免引入新依赖。按 [YAML 1.2.2 规范](https://yaml.org/spec/1.2.2/) 的 JSON 兼容语法书写；仅支持此项目子集，标签、锚点、任意对象构造及隐式类型不在接收格式内。最多读取 65,537 bytes，超过 64 KiB 拒绝。

when 是“数组内任一字典成立”；字典内条件全部成立，字段值数组表示 in，not_ 前缀表示 not-in。字段名、类型和值域封闭，没有 eval 或任意表达式；编译成不可变 tuple / frozenset。effect 为仅含标量的 frozen 模型。判定期间不读磁盘、不联网、不访问数据库。

load_decision_policy 的缺省目录仅用于可信启动 / 开发加载。实际配置解析器须将预期 DecisionPolicyLock 纳入不可变配置锁，并把该锁传给判定器；禁止接收 Worker 自报目录 / 策略 / expected_lock。当前尚未实现配置持久化或服务认证。

## 4. 动作输出与提交责任

FetchDecision 包含 verdict / 全部违规 / primary / 行号 / next_action / job_target / checkpoint_policy / follow_up、覆盖完备性与截断、连接器健康 / 质量标记、恢复预算复核标志、普通输出许可。18 行及 selector 的唯一可执行来源是 YAML，测试的逐行预期来自大纲。

| 行 | 关键输出 |
| --- | --- |
| 1 / 6 | continue；分别 continuation / next_slice，后续稳定身份和作业锁内复核由运行时负责 |
| 2 / 3 / 8 | 覆盖标截断；unknown 完备性仍保持 unknown，不升级成 complete |
| 3 / 5 | periodic 无 continuation 的 slice_end 容忍为正常结局，同时标 periodic_slice_end 质量问题 |
| 7 | 连续流 source_exhausted 保持 partial / unknown，挂连接计时器 |
| 9 / 10 | 限流 cooling；无前缀不推进，有前缀只推进该前缀；恢复绑定冷却代次 |
| 11 / 12 | transport_error 仍经 complete 结束当前任务；recovery_budget_required=true，持久预算耗尽则提交段改 paused，不能无条件写 cooling；行 12 恢复同时绑定检查点版本 |
| 13 | 暂停、不重试、连接器不健康、覆盖 unknown |
| 14–16 | reauth / halt / resync；当前暂停，后续显式授权 / 人工 / resync 路径处理 |
| 17 | 单资源 gone 登记一次缺失意图，内容游标不推进，下一周期再访 |
| 18 | 整个查询目标消失，暂停并告警 |

这些是确定性**意图**。作业 / 周期 / 恢复预算、告警、稳定任务身份、lease、检查点 CAS、I9 观测、I21 计时器、违规受限提交与旧代次控制投影均须由后续事务引擎兑现。本轮不改变 PG、Kafka 或 Worker API 行为，不提供成功空桩。

## 5. PageResult 与签名前接线

classify_collection_page 生成两表版本来自策略锁的 PageResult，以及独立控制报告；纯删除页无需正文也能正常判定。verify_page_result 拒绝 Worker 自报行号、版本或遗漏违规。

`prepare_collection_material` 必须用于后续采集计算路径：先验证 PageResult，再复用 M0-04 的 prepare_material 验证 task / namespace / 根集合 / 永久代次 / 控制前提；结果可交 RuntimeSigner。合法的 violation PageResult 也能生成材料，因为其已接受控制仍需提交。这个入口尚未独立解析证据、引用、引文、详细载荷或实际配置；调用方提供的 verified_roots / epochs / source policies 仍须经平台权威查询。原 prepare_material 保留其有限校验兼容行为，不能绕过新入口签采集包。

已有 PageResult / SubmissionRequest / SignedValidation 的第二批线结构未变。新增 FetchDecision 与 DecisionPolicyLock 的两份快照独立放 `contracts/schemas/v1-decisions/`，此前 29 + 41 份快照保持原字节。

## 6. 完备性证据与重放

```bash
make test-fetch-decisions        # 页面、逐行预期、策略破坏、签名前重算与工具测试
make test-fetch-decision-matrix  # 正式全组合，默认全量，生成 gzip 明细 + summary
uv run --locked python -m scripts.verify_fetch_decisions --ordinal 12345
```

本轮轮廓有 12 个维度，总共 3,686,400 组合；明确排除 921,600 个 single_resource × query_or_stream，不赋 verdict；其余 2,764,800 全部执行正式 matcher，对照**不读取 YAML 的独立逐条 oracle**，核对全部违规、动作行和最终动作。18 行和 19 条均有覆盖。

与 M0-02E 的旧规模探针相比，本轮把“集合内 present 或 failed”拆开：二者在查询 / 流请求的 V8 判定不同。条目结果维度因此从 9 类扩为 10 类；包含未知 outcome 与 missing 的混合反例，以覆盖 V1 / V18 同时成立。正式事实提取与矩阵事实另通过 80 项真实页面逐例核对。旧探针保留其原统计与“非 H11”标识，不替代本轮。

`.cache/fetch-decisions/` 保存每个违规组合一行的 gzip 明细 `[原始笛卡尔积序号, 全部有序违规码]`；首码为 primary，最终动作为 violation。summary 含维度顺序、规则锁、轮廓摘要、计数、序号校验和、gzip 摘要与环境。用 --ordinal 可反解明细，不需一次加载数百万行。显式 --shard-index / --shard-count 会标 exhaustive=false；只有所有分片共同覆盖才能代替全量，不把单片当完整证明。

CI 每个 PR 都跑全量，强于“变更时全量、其他时夜间全量”的最低要求；上传 `fetch-decision-evidence` artifact，保留 14 天，所有内容仅合成枚举 / 序号 / 违规码，无来源正文和标识。大明细不入库，仓库存本机实测摘要 [m0-05a-fetch-decisions.json](../validation/m0-05a-fetch-decisions.json)。失败保留 .partial，不能视为完整输出。

本证据覆盖 bounded_slice 判定规则的有限等价类，不是完整 M0 硬验收、真实来源组合评测或实际作业事务通过。long_session 继续由计划解析层拒绝，启用时需扩规则与轮廓。
