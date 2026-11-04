# M0-05A：正式采集判定器与全组合证据

日期：2026-11-04（Asia/Shanghai），工具 Codex(GPT)。起点 main `cfe37fadc78d846b3c404226722a1c08b6a0ced3`（PR #10），功能分支 `feat/m0-fetch-decisions`。用户要求继续下一项，M0-05 按独立功能 PR 交付；本轮完成 A 判定模块，**没有把整个最小文件闭环标成完成**。

## 1. 背景、依据与范围

大纲 §4.11 明确先执行完整 19 条违规，再匹配 18 行动作；分册一 §1 / §3 要求表版本入配置锁、运行时独立重算；总册 §8.3 / §8.4 要求交付判定器及 bounded_slice 全组合。M0-04 的 prepare_material 仅核对 V1 / V19 和材料前提，M0-02E 的 matrix 仅扫描规模，均不能作为实际调度判定器。

本轮把规则做成可执行、可锁定、可复核的工程组件：正式 YAML、完整判定、可信控制 / PageResult 重算、采集签名前入口、两份输出 schema、独立 oracle / 实际全组合和逐违规 CI 附件。没有增加依赖、业务迁移或运行服务；task / job / 证据表、I5 / I7 / I21、SDK / file 仍待后续功能单元完成。

三级接口影响登记于 ADR 0011、D05-01–08；没有组织签字模拟或 A/B/C 裁剪。独立分支 → 验证 → PR → 最终 head 三项 CI → merge 的授权持续适用。没有 .codegraph/，依约用 rg、不建索引；没有委派子 Agent；使用 ponytail 复用既有模型 / 严格 JSON / 控制校验 / 签名，不引入通用 YAML 库或新规则引擎。所有 shell 经 RTK。

## 2. 开发方案与关键文件

先对照 HLD 逐行翻译动作与全部违规，再建立有限 matcher；使用完全不同的直译 if oracle 做组合核对，避免生产表与测试同源自证。真实页面反例另覆盖生产事实提取；最后将完整重算接到采集材料准备、导出 schema、接入 CI、记录实际结果。

| 文件 | 接手作用 |
| --- | --- |
| `contracts/decisions/fetch-actions.v1.yaml` | 唯一正式 18 行动作，版本 fetch-actions/1.0.0；selector 与 effect |
| `contracts/decisions/fetch-violations.v1.yaml` | 唯一正式 19 条违规，版本 fetch-violations/1.0.0，按编号完整收集 |
| `contracts/fetch_decisions.py` | DecisionPolicyLock / FetchDecision、受限结局约束、独立两模型 registry |
| `contracts/fetch_decision_rules.py` | 严格规则加载、枚举域 / 18 / 19 验证、不可变编译、平台事实提取、唯一行匹配、正式 decide_fetch |
| `runtime/core/collection_decisions.py` | ClassifiedPage、逐项控制独立重算、PageResult 生成 / 自报核对、prepare_collection_material |
| `contracts/schemas/v1-decisions/` | 两份新增快照；原 v1 / v1-second 29 + 41 字节不变 |
| `tests/support/fetch_decision_matrix.py` | 12 维完整 product、10 条目等价类、独立 HLD oracle、分片计数与序号反解 |
| `scripts/verify_fetch_decisions.py` | 默认全量、显式分片、gzip 每个违规组合全部码、摘要 / artifact 哈希 / --ordinal 重放 |
| `tests/unit/test_fetch_decisions.py` | 152 项实际页面、全部行 / 条目、观测集合、错误策略、采集签名前重算和工具反例 |
| `docs/validation/m0-05a-fetch-decisions.json` | 已实际运行的本机全量摘要；包含策略锁 / profile / 统计 / 明细哈希 |
| `Makefile` / `.github/workflows/ci.yml` | 专用测试 / 全量目标；checks 每 PR 跑全量并上传 14 天证据 artifact |
| `docs/contracts/fetch-decisions-v1.md` / ADR 0011 / D05 映射 | 消费语义、版本 / 信任边界、假设与后续责任 |

## 3. 判定核心语义

### 正式事实与信任边界

FetchPageWire 仍是开放枚举接收模型，结构损坏走已有固定 WireDecodeError，尚未形成页面的执行失败仍走 fail。正式上下文沿用 FetchDecisionContext：schedule / bounded_slice、单资源标志、请求集合、已可靠持久化前缀及控制检查结果来自平台，不接受 Worker 自报为权威。

ok 按条目寻址：观测集合为 requested_item_refs；ok 查询 / 列表 / 流：集合为本页 items，且只允许 present；非 ok 有前缀：集合为 persisted_prefix_refs；非 ok 无前缀不得有条目结果。单资源 ok+missing 另为 V18，gone scope 的两种上下文分别判 V16 / V17。未知 outcome 在查询页还不满足“只允许 present”，因此 V1 / V8 可同时成立。

逐条 V1–V19 全部执行，不提前停在未知枚举。forbidden 原因未知同时 V1 / V2；其他 status_reason 是自由诊断文本。codes 排序去重，primary=min；violation 不计算动作。valid 必须唯一匹配，否则固定策略错误，不能默认 idle / 取第一行。suggested_action 不参与规则。

classify_collection_page 先用已认证执行上下文 / 来源授权执行逐项控制校验，然后覆盖 context.control_validation_passed，再计算全部违规。错误信号只留位置 / 固定原因；V1 或一个非法控制不吞掉其他 accepted 信号，供后续受限投影使用。accepted 是校验结果，不是 PG 已应用。

### 配置锁、格式与解释器

采用 YAML 1.2 的 JSON 流式语法 + 文件首部注释，依据官方 YAML 1.2.2 的 JSON 兼容语法；只支持这个项目子集，复用 decode_wire 的重复键 / UTF-8 / 严格类型拒绝。拒绝通用标签 / 锚点 / 任意对象，读取最多 65,537 bytes，超过 64 KiB 拒绝，不在加载错误中回显文件内容。

when 是析取范式：列表内任一字典成立，字典内所有字段条件成立；值数组表示 in，not_ 前缀表示 not-in。只允许 DecisionFacts 封闭字段和准确类型 / 枚举值；没有 eval、用户代码或任意属性。编译后 selector 为 tuple / frozenset，effect 是无嵌套容器的 frozen 模型，运行判定不读文件 / 网络 / PG。

DecisionPolicyLock 含 evaluator_version=fetch-decisions/1、两表版本与两个原始文件字节 SHA-256（含 header / 空白）。已加载策略必须等于传入 expected_lock；同版本改字节也失败。两表内 enum_domains 必须等于正式 FETCH_ENUM_DOMAINS，不能新增枚举而不补规则 / 全量证明。实际服务须由钉定任务配置解析此锁，不能从 Worker 接收目录、policy 或 expected_lock；这个配置存储 / 解析服务当前尚未实现。

### 输出意图与完整材料接线

FetchDecision 显式给行号、next_action、job_target、checkpoint_policy、follow_up、覆盖完备性 / 截断、连接器健康 / periodic_slice_end 质量问题、recovery_budget_required、普通输出许可。unknown 完备性不因预算截断升级为 complete。periodic 的 slice_end 无 continuation 容忍为行 3 / 5 并记质量；有 continuation 为 V12。

行 11 / 12 输出 cooling **意图**并要求真实恢复预算复核，预算耗尽由提交段改 paused；新任务不重置预算。恢复身份、cycle / 冷却代次、检查点版本、作业 fencing、告警与计时器写入均由后续引擎负责。行 17 只有一次缺失观测意图，非真实 I9 写入。不能把本轮 DTO 当作事务已完成。

verify_page_result 独立重算并拒绝伪造行号、表版本或漏报违规。后续采集计算路径使用 prepare_collection_material：验证完整 PageResult，再调用已有 prepare_material 核对 task / 模式空间 / 根集合 / 永久代次 / 控制前提，之后才交 RuntimeSigner。测试同时验证 valid 和合法 violation 都能生成 / 验证 HMAC，漏 V9 的包在签名前失败。

**尚未实现**：证据 / 引用 / quote / 详细 Claim / 文档 schema 校验、独立血缘与实际代次查询、实际配置锁存储。verified_roots / epochs / policies 仍须调用方从平台权威解析。原 prepare_material 兼容保留其有限行为，不应直接作为采集入口。本轮没有公开服务或签名端点，也没有真实密钥。

## 4. 全组合方案与实际证明

旧 M0-02E probe 合并了集合内 present / failed，而查询请求下两者的 V8 结果不同。本轮拆开，条目维度为 10 类：空、内 present / failed / missing、外 present / missing、内 / 外 unknown、内 / 外 unknown+missing。未知码和 missing 可同页，覆盖 V1 / V18 同时命中。

12 维轮廓：schedule(2) × status(12) × status_reason(5) × failure_scope(4) × completeness(4) × end_reason(6) × continuation(2) × prefix(2) × single(2) × outcome_case(10) × request_mode(2) × controls_valid(2)。

- 原始 / 选中：**3,686,400**。
- 唯一不可达排除 single_resource × query_or_stream：**921,600**，不赋 verdict，正式模型另有真实拒绝反例。
- 实际判定：**2,764,800**；valid **4,685**，violation **2,760,115**。
- 每个组合的全部违规码与动作行都和不读取 YAML 的独立 oracle 一致；valid 另比最终 next_action，零 / 多动作匹配由生产代码拒绝。
- 全部 18 行均命中、全部 V1–V19 均命中。完整 per-row / per-code 次数在已提交摘要，不把多违规计数之和当组合数。
- 原始可达序号校验和 **5,096,058,624,000**，profile digest `106e1c9b1af79a1d162b52ba14efbeeb7dba76f3478a24b7e89d3ddbaf8ca6a6`。
- 本机 Darwin arm64 / Python 3.12.12，扫描及 gzip 输出共 **30.945 秒**；不包含真实网络 / 数据库 / 作业提交。

本机完整明细 `.cache/fetch-decisions/full/violations-0-of-1.jsonl.gz`：2,760,115 行，8,176,456 bytes，SHA-256 `33c6336fd17580a5579f2f6defec85bf7cee4a873e2492887633e6721bfa16e0`。每行 `[raw_product_ordinal, all_ordered_violation_codes]`，首码 primary，最终动作为 violation；summary 记录维度顺序 / 策略锁 / 环境，--ordinal 可反解。gzip 的时间元数据使不同运行的文件摘要可能不同，CI 附件以其自己的 summary 为准。

大明细不入库，仓库存实测 JSON 摘要；测试强制摘要的策略锁 / profile 与当前规则一致。CI checks 每个 PR 都默认全量，不用抽样冒充全量；artifact 名 fetch-decision-evidence，保留 14 天，只有合成序号 / 枚举 / 违规码。显式分片摘要标 exhaustive=false，ordinal 分片全体合并才覆盖全集；单片可能偏向某些维度，不作为完整证明。失败文件为 .partial，不能宣称完整输出。

## 5. 验证命令与实际结果

```bash
rtk proxy make test-fetch-decisions
rtk proxy make check
rtk proxy make test-fetch-decision-matrix
# 有意修改表后更新本机摘要，必须审查规则和统计 diff：
rtk proxy uv run --locked python -m scripts.verify_fetch_decisions --output-dir .cache/fetch-decisions/full --summary docs/validation/m0-05a-fetch-decisions.json
rtk proxy uv run --locked python -m scripts.verify_fetch_decisions --ordinal 12345
# 有意改模型后生成，普通检查不修复漂移：
rtk proxy make schema-write
rtk git diff --check
```

专用最终 **152 passed**，包含 18 行独立预期、19 条各自触发、多违规全部命中、不执行动作、80 项实际观测集合提取、纯删除、独立控制 / 伪造 PageResult、完整材料 / 签名、零 / 多行动作、错误策略 / 类型 / 重复键 / 同版本篡改、模型 / 工具 / 分片反例。早期测试误用 pytest 保留参数名 request 已改 request_mode；复用第一批 page() 自带占位控制，独立测试 helper 已明确清空再按需添加，没有修改生产语义迎合样本。

完整 **make check：724 passed / 48 deselected，11.39 秒，收集 772 项**。八条导入边界、连接器静态检查、事务静态辅助（当前仍 0 业务实现）、迁移 1 revision、四组 Schema **3 + 29 + 41 + 2** 均通过。补充摘要漂移断言后专用测试再次通过；正式规则 / matcher 在本机全量后未改动，不重复跑相同全量。此前两批快照 git diff 为空。

6 个新 Python 文件具有最顶部 Codex(GPT) / 本地日期 header，41 个函数均有说明；两个 YAML 数据文件亦有 AI header，JSON 快照与实测摘要不加注释。9 份相关 Markdown 本地链接、AST / header / 函数说明和 git diff --check 通过。没有修改 uv.lock、应用依赖、Compose 或业务迁移；本机未重跑无关 PG / Kafka / cgroup 故障，远端最终 head 的 infrastructure / isolation 必须实际通过，不能以本轮纯判定测试替代。CI 新增官方 upload-artifact@v4，依官方说明只对专用合成目录 `.cache/fetch-decisions/` 开 include-hidden-files，避免默认隐藏目录过滤导致证据未上传；矩阵实际运行但附件缺失会判错。

## 6. 后续接手与回退

1. 读本 memory、消费规范、ADR 0011 / D05、roadmap 和 M0-04 的材料 / 锁约定；真实来源 G 的资格与保证开放项仍有效。
2. 看 YAML / models / matcher，跑专用测试；看 independent oracle / profile，核对本机摘要并反解一条多违规记录。任何规则 / 枚举变化都更新版本 / 锁与实测摘要，再跑全量，不能只改 expected。
3. 后续单元先补闭环所需实际载荷与配置锁（Claim / CloudEvents / Evidence / 文档 / Manifest / 任务），再接真实证据 API、控制 / 任务 / 作业 / 定时 DDL 与 I5 / I7 / I21；事务内接口没有 Fake。
4. 实际采集计算入口必须用 prepare_collection_material，并提供从任务 / 配置 / 证据服务解析的前提；再签名交提交段。按本轮行意图实现预算 / CAS / 后继身份 / 计时器，不能把 job_target 当无条件更新指令。
5. 最小运行时 / SDK / file / 准入 / 明确 I8 注册 Fake 接线，验证文件一页 → 提交 → 检查点 → 事件；未收到注册确认不得 lease。M0-06 再接真实注册 / 撤回 / I9 / 派生屏障与故障矩阵。当前未达到整个 M0-05 集成点或 M0 出口。

有限规则证明不覆盖任意真实载荷 / 引用 / 来源 / 资源耗尽，也不是完整服务接线后的 H11 验收；long_session 继续在计划层拒绝，启用时扩规则和组合。真实采集、模型调用和许可未开启。

回退撤销本轮规则 / 新入口 / 两份 schema / 工具和 CI 全量步骤，保留 M0-04 原基础 helper；无业务数据变更。真实配置或消费者出现后，表、解释器、语义或锁变更须显式版本化，不静默改义。

## 7. PR 与交付

PR 创建后在同一记录补实际链接 / 实现提交；最终受检 head、CI run、artifact 与 merge 证据以 PR 实际交付记录为准，合并前须确认 checks / infrastructure / isolation 全成功。PR #10 的实际最终 head / CI / merge 已补到原 memory。

GitHub 连接器已重新发现支持 PR 创建 / 合并等能力，优先使用；本地 git 推送按已有 kai2dot0 授权操作，不切换全局 gh 默认账号，不打印或落盘凭据。若连接器权限不足则按实际错误说明并使用已授权 gh 后备。禁止直接提交 / 推送 main。
