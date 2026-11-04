# ADR 0012：采集配置冻结与具体载荷绑定

- 日期：2026-11-04；状态：工程采用，未模拟组织签字。
- 依据：大纲 §4.1 / §4.3 / §4.4 / §4.6 / §4.7，总册 §5 / §7.3 / §8.2–8.4。
- 总册 §11.1：三级接口影响，新增配置 / 事件 / 证据 / 任务契约及签名前入口；没有旧业务消费者或数据迁移。不改变 HLD 职责，不实施 A/B/C 裁剪。

## 决定

M0-05A 能判定页面，但不能证明其证据、事件、配置与实际任务一致。B 先完成这些输入与具体校验，真实事务随后独立交付。共享模型继续采用已有 Pydantic / JCS / SHA-256，复用严格 JSON 解码与稳定引用，SDK 和平台共享输出 ID 实现；不新增依赖或表达式引擎。

Manifest 的可继承预算由显式、有版本的池缺省展开，完整有效配置连同上下文、来源、事件、决策和实际 Schema 内容进入锁。比较受信引用，再逐字段重建任务；只相信自报版本或自报内容摘要不足以绑定任务。平台角色 / 来源资格快照是前置条件，模型不实现认证。

事件拆为结果包语义意图与平台外发记录，再显式转换为 CloudEvents JSON。采用 [CloudEvents v1.0.2 规范](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) 与 [JSON 格式](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/formats/json-format.md) 的必需上下文、顶层扩展标量与 null 处理；项目额外限制 JSON 对象、引用式来源内容、内联元数据尺寸、父事件上限及 subjectseq 编码。内部 Schema 不冒充外部线格式。版本采用 [SemVer 2.0.0](https://semver.org/) 的核心 / 预发布规则；领域包版本范围仍等后续加载器解析，当前非空依赖明确拒绝。

证据身份来自不可变逻辑清单，字节摘要和治理版本分离。签名前从可信证据目录独立取根，核对输出载荷、来源、flags、检查点与判定；发现撤回内容但尚无存储剔除器时拒绝签普通内容。违规页仍允许独立合法控制，不因被忽略的普通意图根而外层拒绝。

## 当前编译 profile 与后续影响

运行配置只装配内建 arrival 元数据事件；filter 为路由字段等值，partition_key_expr 是路由属性名。通用 codec 的 causal / attr* 表达能力不代表 registry 服务或别名映射已实现。side_effects 当前限 none / internal_write / external_read，不开启外写。Demo max_batch=1、worker_checkpoint=false；file 不联网。缺少实现的受控导入、Null、依赖解析、后继载荷和剔除记录均明确报错。这些是分阶段实现边界，没有从最终需求删去功能。

新增 v1-collection 独立 16 根 Schema，旧三批正式快照保持不变。没有 SQL、网络端点或 lock 顺序变动。回退可移除新模型 / helper / 快照 / 测试并恢复共享函数调用；没有业务数据可回滚。真实消费者出现后，身份、字段语义与 profile 扩展必须显式版本化并做兼容评审。完整调用与未实现项见 [消费规范](../contracts/collection-config-v1.md)。
