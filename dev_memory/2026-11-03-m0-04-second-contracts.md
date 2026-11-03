# M0-04：第二批契约、语义投影 / 签名与同事务上下文

日期：2026-11-03（Asia/Shanghai），工具 Codex(GPT)。起点 main `c553e134b9d3655da6170920dbb205fbb948d6ca`，PR #9 已合并；独立分支 `feat/m0-second-contracts`。用户授权继续下一项，按 roadmap 交付 M0-04；本记录足以供后续 Agent 复现及接入 M0-05，不代表完整系统已能运行业务。

## 1. 背景、依据与范围

第一批有执行上下文、受控传输和开放 FetchPage，但没有提交包、控制结构、摘要投影和签名；二组也缺少可依赖的事务上下文。总册 §3 / §8.2 要求第二周补齐完整提交和控制协议，§7.4.1 明确事务内接口的同连接 / 不开事务 / 不网络等待及锁序；分册一 §1 / §3 / §4、分册二 §2 / §5 / §6 与大纲 §4.2 / §4.3 / §4.7 / §4.11 / §4.12 是语义依据。

本轮交付 41 个新公开根模型 / schema，I1 / I5 / I7 / I8 / I9 / I10 / I14 / I15 / I17 / I19 / I21 相关接口结构，实际共享投影 / JCS / SHA-256、运行时 HMAC、逐项控制校验、同事务守卫 / PG facade 及真实事务实验。I16 平台控制入口按总册 M0 末安排，当前无状态转换实现。

范围是“可校验契约及可执行基础组件”，不把协议类当服务，不实现任务 / 控制 / 证据业务表、完整 13 步数据库引擎或所有适配器。没有真实来源、模型调用、密钥、生产数据或数据库迁移。按总册 §11.1 三级接口影响登记，见 ADR 0010 / C04；没有 A/B/C 裁剪或组织签字模拟。继续采用每模块功能分支 → PR → 最终 head 三项 CI → merge，不直接提交 / 推送 main。

没有 `.codegraph/`，依约使用 rg、不创建索引；没有子 Agent。使用 ponytail 保持现有叶子层、Pydantic / schema 工具、锁守卫、PG 实验 fixture，没有新增依赖。shell 调用通过 RTK，CI / Make 不依赖本机 RTK。

## 2. 开发方案与实际顺序

1. 对照 HLD 第二批冻结清单，先列全结果输出类别、操作 / 分段 / 回执键、材料绑定、控制对象字段与 I5 / I7 / I9 / I10 / I21 锁层级。
2. 复用 ContractModel / contract_field，建立 control / state / bundle / worker / transactions / validation 结构，再补 withdrawals / availability / normalization / quality 的第二周接口。
3. 保持 contracts 与 spi 独立叶子：SPI 用泛型 Protocol，SDK 绑定 Worker 类型，runtime 绑定平台事务类型。不放宽 import-linter，不把 signer 放进 SDK。
4. 把原 JCS 原样移到共享 contracts 层，保留 runtime 兼容重导出；实现精确类型 + 字段政策投影，固定外层 lease_token 排除白名单。
5. 实现完整材料 HMAC、独立期望绑定校验、私有本地密钥文件读取；平台材料 helper 只核对已独立解析前提，不冒充完整业务计算校验。
6. 实现逐项控制结构 / 权限校验、控制锁计划、十三步 / 根断言 / 生命周期守卫和真实 PG 借用端口。复核时把原“SQL 后登记”改为 acquire_lock 的“先验顺序再执行 SQL”，避免等锁后才发现逆序。
7. 增加第二注册表，生成独立 41 schema，第一批 29 快照 diff 为空；将静态辅助门禁、单元反例和 PG 实验接进既有 Make / CI。
8. 完整验证，更新规范、ADR / 规则映射 / README / roadmap / memory；功能分支提交 PR，核对最终 head checks / infrastructure / isolation 全绿后合并。

## 3. 核心文件与接手入口

| 路径 | 作用 |
| --- | --- |
| `contracts/control.py` | 完整控制身份、根 / 永久代次、三种控制信号、逐项校验、固定来源比较基准、控制对象 / 原因 / 覆盖证明 / 保持 / 交接 / gap / 账号快照 DTO |
| `contracts/state_service.py` | I19 完整地址 / 显式读结局 / 版本、状态写意图与命名空间纯判断 |
| `contracts/bundle.py` | 全类别 ResultBundle、LogicalWrite / EvidenceCreation / 派生 / 任务 / 检查点 / PageResult / 剔除，声明根汇总与终态纯映射 |
| `contracts/worker.py` | I1 lease / heartbeat / submit / fail、TaskBinding、各结局、普通 / 投影回执键与分段规则 |
| `contracts/transactions.py` | 锁请求、I5 / I7 / I9 / I10 / I21 mutation 与非提交回执 AdapterResult |
| `contracts/validation.py` | ValidationMaterial / SignedValidation、完整永久代次集合与有序来源锁 |
| `contracts/canonical_json.py` | 原 RFC 8785 实现迁移至共享叶子；`runtime/core/canonical_json.py` 保留兼容导出 |
| `contracts/semantic.py` / `sdk/semantic_bundle.py` | 实际逐字段投影与 SHA-256；SDK 复用同一入口 |
| `contracts/primitives.py` | contract_field 可声明 include / exclude；validated_snapshot 从原生值重验，避免 NaN 经 JSON 变 null |
| `runtime/core/validation_signatures.py` | prepare_material、独立控制 / 根集合一致性、来源配置锁、HMAC 域 / key 绑定、受限密钥文件与验签 |
| `contracts/withdrawals.py` | I8 uint64 连续日志、清理证据 / 注册 / 前沿 / 退役 / 待核验副本 |
| `contracts/availability.py` | I17 逐引用可用性结局及阻断原因 |
| `contracts/quality.py` | I14 六个唯一维度、computation 特例、Claim 摘要 / 规则包 / 目录版本绑定 |
| `contracts/normalization.py` | I15 纯输出结构、observed 关系、exited_query 不作删除、不允许 tombstone |
| `spi/submission.py` | 独立泛型协议、SubmissionContext / TransactionExecutor / SqlResult；没有数据库工厂或 Fake |
| `sdk/worker_types.py` / `runtime/transaction_types.py` | 分别组合 Worker 与平台类型，Worker 不接触事务 |
| `runtime/core/submission_context.py` | COMMIT_STEPS、最强锁计划、先验锁序 / 步骤再 SQL、根 / 来源前置条件、调用范围和失败失效 |
| `adapters/submission_transaction.py` | 只借已有 PG 事务、backend / txid 绑定、单 prepared SQL / 行数预算 / 固定错误与 SQLSTATE |
| `contracts/second_registry.py` / `contracts/schemas/v1-second/` | 41 个公开根模型与自动 schema，独立于第一批 |
| `scripts/check_transaction_adapters.py` | 未来 `adapters/transactional/*.py` 的网络 / 连接 / 提交静态辅助检查；当前明确 0 实现 |
| `tests/support/second_contracts.py` | 纯合成输入，复用第一批 fixture；不是证据解析 / 存储 Fake |
| `tests/unit/test_second_contracts.py` | 41 项消息 / 控制 / 来源配置 / 状态 / 退役 / 插件契约反例 |
| `tests/unit/test_semantic_signatures.py` | 36 项投影独立 oracle / 篡改 / 默认 / NaN / key 文件 / 跨模式绑定反例 |
| `tests/unit/test_submission_context.py` | 17 项步骤 / 锁 / 根 / 作用域 / 静态门禁；RecordingPort 仅记录调用 |
| `tests/integration/test_submission_transaction.py` | 9 项真实 PG 同事务 / 原子性 / 生命周期 / 错误脱敏实验 |
| `docs/contracts/second-batch-v1.md` / ADR 0010 / C04 | 完整消费规范、决策依据、三级影响、未完成项 |

## 4. 必须保留的设计细节

### 摘要与签名

投影只接精确 SubmissionRequest，默认展开后递归读取已声明模型字段的 digest 元数据；固定路径 `lease_token` 是唯一排除。动态 JSON 中同名 lease_token / receipt_id 始终是语义事实。所有稳定对象 ID / revision / digest、上游事件时间、控制内容、任务 / 操作 / 段前沿、剔除和有序早期段摘要都参与。其他排除类别没进入模型，不能扩张成全局键名过滤。

ContractModel 的 frozen 仅冻结顶层属性。发现把嵌套可变字典直接 dump JSON 可能使 NaN 变 null，故用 model_dump(mode=python) → strict model_validate 再投影 / 签名；NaN 和宽整数反例验证此边界。共享 JCS 位置变更不改变算法，原 88 项 JCS 测试继续回归。

HMAC 域为 `april.validation.v1` + NUL，JCS 对象同时含 algorithm / key_id / material。材料含完整 TaskBinding、操作 / 段 / 前沿、摘要、独立根、永久代次、控制检查、来源锁、剔除。验签核对独立期望绑定和当前请求重算摘要，不能用 Worker 自报期望值，也不能替代后续租约 / job fencing 的锁内检查。密钥文件当前用户私有普通文件，拒绝 symlink / group-other 权限 / 无效大小；测试 key 是公开 fixture，仓库没有真实密钥。

prepare_material 只验证“已独立解析前提”的一致性，当前没有真正查询证据 / 引文 / Claim schema / 血缘，也未做完整 V2–18。V1 未知枚举和 V19 非法控制独立核对，合法控制不因其他项或未知页状态丢失。signer 不能作为“任何 Worker 包可请求签名”的公开服务。

### 提交 / 锁 / 控制

普通 complete 回执 `(task, complete)`，checkpoint `(task, checkpoint, segment)`；旧代次投影独立 `(task, control_projection, lease_generation, digest)`。实际引擎必须优先区分旧代次终态再做普通摘要冲突，投影在同一已持任务锁事务内验签 / 登记合法控制，不能再开事务等自己。Demo checkpoint unsupported；complete 段为 after+1，并绑定全部早期段摘要有序列表。

control / state 冲突保留租约，分别返回最新永久代次 / 状态版本；前者不耗普通尝试，后者耗独立冲突预算。前沿不匹配规范为 conflict 原因、动作 re-lease；checkpoint CAS 失配标 lease_lost + 诊断。blob_condemned 重传在事务外、同语义包重提，不在第八步改绑实例。

十三步只是本轮守卫的挂点。第六步文档 / 账号 → 来源 → 交接账号快照 → 撤回计数器；第八步 blob → logical；第九步 state → observation → timer；第十步 derived；第十一序号、第十二 job。预先合并控制模式：引用 KEY SHARE、临时 NO KEY UPDATE、永久或合并 UPDATE，不升级。来源 authority_position 账号控制用 SHARE，切换 / 接任用 NO KEY UPDATE，来源级记录不能用 SHARE 冒充写锁。

acquire_lock 先检查当前步骤、接口允许层、预选模式和锁序，再执行借用事务 SQL；SQL 目标 / 键 / 模式的真实对应仍须可信实现审查。异常关闭上下文，owner 整事务回滚。每次 adapter 重新 assert_roots 或来源级 assert_source_locked；离开范围后保留端口失效。facade 无 cursor / connect / commit / rollback；真实 txid 不同即拒绝。它不是恶意 Python / SQL 沙箱，静态检查也不是物理网络隔离。

控制 DTO 为后续 DDL 提供列清单，尚未创建业务表：完整主键；availability_version 与永久撤回 epoch 分离；原因清除后保留位置 / 时间；hold 保存每次冲突类型 / 时刻与预算、关闭依据；同账号 / 原因未关闭部分唯一；每 hold 至多一在途回访；handoff / 多条 gap / 账号快照独立表。永久撤回只 doc / account，source 只有临时读门控；大纲 §4.3 的 source 泛称按 §4.12 / 总册明确锁语义解释，变更须版本化。

### I8 / I17 / I19 与插件

日志序号是 uint64 十进制字符串，页从 after+1 连续，广播只唤醒；注册需真实缓存清理且成功后才 lease，frontier 仅已处理连续前缀，无法核验退役保留 pending 副本。当前模型引用 / bool 不能证明 OS 或 PG 事实。

I19 无 Worker 直接写 API，写只在包第九步 CAS；found 必须有版本，null 可以是值，拒绝 / unsupported 不带内容。I17 当前可用性按引用返回明确原因，不含正文，实际根查询 / 控制 / 文档核验 / pending 撤回待服务实现。

normalizer 无网络 / DB 参数，不产 tombstone，关系只能 observed，exited_query 不追溯删除。ValidationRecord 六维恰好唯一且含 computation，其他五个名称 HLD 未指定，本轮留钉定 dimension_catalog；fixture-a…e 不是产品语义，实际目录和规则加载尚待实现。没有把 pending / 不适用标为通过。

## 5. 验证命令与实际结果

```bash
rtk proxy make schema-write                 # 仅有意更新模型后生成，审查 diff
rtk proxy make test-second-contracts
rtk proxy make check
rtk proxy env TEST_DATABASE_URL=postgresql://april:april_local_only@127.0.0.1:15432/april make test-integration
rtk git diff --check
```

第二批专用单元最终 **94 项**（41 契约 + 36 摘要签名 + 17 守卫）全部通过；初轮 91 项后补入三个固定来源比较基准的锁模式 / 配置锁反例。真实 PG 集成 **33 passed / 584 deselected，2.42 秒**，执行时尚未加入这三个纯单元测试，含既有 24 项和本轮 9 项。新 PG 实验每项只建随机 schema，关闭自有连接后删除自己的 schema，不动业务表 / 有价值数据库；实际使用项目专用本机开发 PG16。

PG 实验包括：同 backend / txid；独立 observer 提交前不可见、提交后可见；显式异常整事务回滚无残留；COMMIT / BEGIN / SAVEPOINT / ROLLBACK 和多语句拒绝；不活动 / 新事务无法复用旧 facade；超行数预算失效；服务器原始 SQL / 参数不回显。只验证受限基础端口，不计为 I5 业务实现。

最终完整 **make check：572 passed / 48 deselected，10.54 秒，收集 620 项**；包含全部第二批 94 项。schema 检查为历史 3 + 第一批 29 + 第二批 41；第一批目录 git diff 为空。八条导入边界、迁移 1 revision、连接器静态门禁通过；新事务适配器静态门禁明确输出 0 implementation files。26 个新 Python 文件的 AI header / AST、150 个函数 docstring、8 个相关 Markdown 的本地链接与 git diff --check 均通过。

本机首次未提权 uv 触发 macOS SystemConfiguration 空对象 panic；以现有许可在沙箱外运行后测试正常，不把崩溃说成测试失败或跳过。没有变更依赖、锁文件、Compose、迁移或 CI 拓扑；Kafka / isolation 在 PR 的实际 CI 验证，不把本地未重跑说成通过。

## 6. 接手顺序、开放项与回退

1. 先读本 memory、second-batch-v1、ADR 0010 / C04、roadmap，再读第一批规范与 G 来源登记。
2. 看 worker / bundle / control / semantic，运行专用 94 项；随后看 RuntimeSigner 的信任前提，不直接暴露 sign / prepare_material。
3. 看 submission_context / spi / PgSubmissionTransaction，重放专用 PG 实验；未来业务适配器放 `adapters/transactional/` 接静态检查，不能使用 Fake 绕过同事务 / 根锁。
4. M0-05 先完成首闭环需要的详细 Claim / CloudEvents / Evidence / 标准文档 / Manifest 钉定 schema、解析 / 引用验证、完整 V1–V19 / 18 动作表，再建最小任务 / 作业 / 证据 / 控制表和真实 I5 / I7 / I21、SDK、文件连接器、明确注册 Fake。按职责独立 PR，可再切小交付，不把本模块 envelope 当详细业务 schema。
5. 文件一页 → 原始证据转正 → 控制 / PageResult → 检查点 CAS → outbox / 事件是首个真实集成点；M0-06 才替换真实注册、撤回 / 派生屏障、I9、监督器与故障 / 删除矩阵。不能把 620 项当前收集数当 H1–H17 全部通过。
6. I16、控制比较 / hold / handoff / resync / coverage proof、六维目录成员、签名轮换 / key_epoch、租约 / 收口 / 旧代次投影、状态冲突预算、真实可用性服务均未完成；真实来源仍禁采，G 的许可 / freshness / 送达上界保持开放。

回退无业务数据迁移：撤销本轮新模型 / SPI / alias / schema / 门禁，将共享 JCS 移回原 runtime 路径并恢复兼容导入与检查组。新消费者实际接入后禁止直接修改字段 / 摘要 / 签名 / 回执键语义，须显式版本与支持期。

## 7. PR 与交付证据

本模块 [PR #10](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/10)，实现提交 `679b88298f6be51510c61acdf7e69561a7f97c5e`。本次关联补记也经同一功能分支提交；合并前核对**最终 head** 的 checks / infrastructure / isolation 三项均成功，实际受检 head / run 及合并证据以 PR 交付记录为准，不能复用旧 head。原 PR #9 的实际 CI / merge 已补入其原 memory；本轮没有直接向 main 提交 / 推送。

继续保持全局 gh 默认账号 kylechandev；授权 kai2dot0 凭据仅在子进程内用于本仓库 git / gh，不打印、写盘或重新申请。所有发布操作沿用户已授权的 PR → CI → merge 流程。
