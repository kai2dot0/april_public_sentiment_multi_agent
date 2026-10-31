# ADR 0009：第一批正式契约 v1 的工程实现

日期：2026-10-31，Asia/Shanghai。对应 roadmap M0-03，依据总册 §3、§8.2 / §8.4；分册一 §1、分册二 §2 / §3 / §6；大纲 §2.4、§4.6、§4.7、§4.9、§4.11、§8.2。

## 交付与边界

将已验证的 Pydantic / 严格 JSON 工具用于正式执行上下文、传输消息、FetchPage / 条目结果、状态错误处理和 ID 四元组输入。29 个公开根模型注册到 `contracts.registry`，派生 JSON Schema 进入 `contracts/schemas/v1/`；原 3 个探针 schema 保留独立路径和 API 默认行为。调用规范见 [first-batch-v1](../contracts/first-batch-v1.md)。

本轮是新增跨组契约的三级影响记录；没有已有正式业务消费者 / 业务数据，因此无旧数据迁移或 SDK 兼容返工。HLD 中“双方接口负责人签字”的组织流程没有被模拟；本 PR / memory 保存可审查工程基线，不声称指定人员已签字。不新增第三方依赖、不改变事务 / 锁序 / 权限底线，不做 A/B/C 裁剪，不修改 HLD 原件。

完整提交 envelope、结果包投影 / 签名、事务内接口、独立控制契约与状态服务仍属 M0-04。18 行 / 19 条 YAML 与纯函数判定器在后续运行时接入中交付（总册 §3 为 M0 内），本轮不把 V1 路径识别当 H11 完成。不实现真实 I4 服务、受控代理、鉴权、配额消耗、记录持久化、SDK Worker 循环或连接器。

## 选择与理由

1. **复用唯一严格字节入口**：沿用 WireModel / decode_wire / field_metadata / checked_schema，避免把新模型接入 FastAPI 默认解析后丢掉重复键或未知枚举。正式 ContractModel 仅补安全 JSON 数值与属性冻结，不新建第二套解析器。frozen 只限制顶层属性赋值，嵌套容器并非权限边界；不可信输入仍须在平台侧重新校验。
2. **线模型与构造模型分开**：FetchPageWire 的页面枚举用 str；FetchPage 的构造侧用 Literal。未知 status / outcome 与 tombstones 一同保留；组合违规由后续统一判定器处理。控制数组本批是保留内容的 JSON 对象；第二批独立控制校验及权限通过前绝不写入。避免先行强类型控制解析导致一个非法控制项把其他合法墓碑一并丢掉。
3. **数值域一次确定**：所有计数 / 代次使用 JCS 安全整数，外部位置用字符串；JSON 扩展中的宽整数也拒绝。原 int64 探针未改义。时间使用严格 UTC 字符串，保持精度；不把缺失证明展开成当前时间或空“成功”。
4. **保持叶子依赖边界**：不修改现有 import-linter。SPI 用 stdlib 泛型 Protocol 声明行为，SDK type alias 组合 contracts 具体类型。contracts 与 spi 无相互依赖，也无 runtime / adapter / app 依赖；Protocol 无可实例化实现，不返回空成功。
5. **接收登记边界显式化**：连接器公开接口只收帧引用与上游事件身份；SDK 才能附接收时刻 / connection / seq / catch-up。时间和认证仍必须由真实服务依据可信帧核对，消息形状不是可信证明。I4 不开放原始 socket；提供 WS send / receive / close 和有依据的 caught-up 确认。
6. **模式和预算在构造阶段可验证**：五种模式、切片导入 / 复现、显式 io 配额与传输预算、长会话拒绝、continuous 恢复能力、真实来源 guarantee 闸门都有反例。平台组装实际有效配置及认证将在后续接入，不能把 worker 自报字段当权限。
7. **ID 先固定材料而非随机分配**：域分隔 + 四元组 JSON 数组明确排序、数值与 UTF-8；无裸拼接、随机性、平台 hash 或字符串归一化。SDK 实际哈希 / ID 输出实现稍后复用材料，既有对象修订不重新派生。

发布策略具体名称补为 automatic / manual_only，前者仍受平台审批策略限制；不代表任意自动发布许可。HLD 中段前沿不匹配同时出现独立结局与 conflict 原因写法，本批明确两种表述的处理均为重新 lease，提交响应 envelope 在第二批收敛。相关登记见 [规则映射](../rule-map/m0-first-contracts.md)。

## 验证和交付

新增第一批契约套件覆盖 schema 漂移、模式 / 配额越权、计划能力、未知值保留、控制载荷、完整数字和默认值、位置域、ID 材料、错误原因 / 结局一一对应、接收时间伪造、URL / header、Base64 / UTF-8、中断审计及真实 loopback HTTP。通用 HTTP 生命周期从既有测试提取复用，原 43 项探针测试不改义。没有生产测试路由或公网流量。

最终测试数、CI、PR 见本模块 [dev_memory](../../dev_memory/2026-10-31-m0-03-first-contracts.md)。CI 继续运行 checks / infrastructure / isolation，必须最终 head 三项全成功再合并。Schema gate 同时覆盖正式和探针；不会自动改写快照来掩盖差异。

无数据库迁移、凭据、新网络目标或业务数据改动。回退时撤销新增模型 / SPI / SDK 类型绑定 / 正式快照及入口，恢复仅探针 schema gate；实际出现消费者后必须按支持期与兼容映射演进，不能原地更改 v1 字段 / ID 材料。后续将模型提升到实际入口时必须补认证、受控网络、流式资源限制及业务事务测试，不能以 Schema 或顶层 frozen 替代。
