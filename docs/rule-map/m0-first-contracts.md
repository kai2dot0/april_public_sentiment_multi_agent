# M0-03 第一批契约：规则映射与兼容性登记

新增契约按总册 §11.1 三级影响留痕；没有旧业务消费者、业务数据迁移或原公开接口破坏。下表是仓库工程记录，不是指定角色的签字表。无 A/B/C 裁剪，未修改 HLD 原件。

| 编号 / 依据 | 实现 | 验证与边界 |
| --- | --- | --- |
| C03-01：总册 §3、分册一 §1 | 29 个根模型唯一注册表 / v1 JSON Schema、逐字段元数据、显式默认 | 生成 / 漂移 / 嵌套属性检查；原探针保留；组织签字未模拟 |
| C03-02：大纲 §4.6 | 五种执行模式、formal / isolated、显式授权和配额、切片来源命名空间 | 隔离重采 / 外部副作用 / 漏配配额拒绝；切片导入 manual_only；数据模型不代替平台认证 |
| C03-03：大纲 §4.11、分册二 §6 | FetchPageWire / FetchPage、ItemResult、ObservationEvidence / Coverage / Checkpoint | 未知枚举 / 空白原值和控制 JSON 保留；结构损坏拒绝；V1 路径不代替 V1–V19 判定 / 受限投影 |
| C03-04：总册 §3 / §6.8、分册二 §7 | 位置域、排序域、游标语义、switch / takeover 报告及 accepted resume 起点字段 | 外部 seq 字符串精确保留；未知起点为 null；报告不自行建立 handoff，持久重复携带稍后接线 |
| C03-05：大纲 §4.7、分册二 §2 | 状态域、冲突 / retry / fail / submission 的处理语义 | enum 与处理表对应；未知 / 缺原因不重试；control 与 state 重算预算不同；具体提交响应结构在第二批 |
| C03-06：大纲 §4.7 的前沿不匹配写法差异 | 独立 segment_frontier_mismatch 与 conflict 原因均映射 re_lease | 只消除 SDK 动作歧义，不提前决定响应 envelope；第二批继续同记录落实，不改变恢复语义 |
| C03-07：总册 §3 I4、分册二 §3 | 传输 / Socket / 准入 / 实时登记 / 运维指定的泛型 SPI，SDK 绑定具体消息 | 既有八条导入边界仍通过；Protocol 不实例化，无成功空桩；真实服务后续交付 |
| C03-08：分册二 §3 接收记录 | 连接器 RecordReceiptCommand 与 TrustedReceiptRegistration 分开 | 时间 / connection / seq 注入被拒；已登记必须有 durable earliest 元数据；未登记无时间证明；不宣称已持久化 |
| C03-09：大纲 §2.2 / §4.6 / §4.11 | 绑定 grant、io 配额 / 预算、URL / header / bytes 形状 | 预算扩大、凭据 / 路由头、userinfo / fragment、非标准 Base64 / 文本非 UTF-8 拒绝；代理 / DNS / TLS / 总预算执法尚未实现 |
| C03-10：分册二 §3 能力闸门 | 有效 guarantee 只允许 http-fixture / test；long_session / 不支持恢复拒绝 | 不靠配置启用真实来源接收时间证明；完整 Manifest / 配置解析仍待后续 |
| C03-11：总册 §3 ID 输入、大纲 §4.7 | 安全整数域 + 原样 UTF-8 + 域分隔四元组材料 | 固定向量、重传一致、不同段 / 类型 / 索引分离、Unicode 不归一化；SDK 哈希输出稍后实现 |
| C03-12：大纲 §4.9 | 正式 / 探针快照隔离、首次 v1 ABI、原 schema 工具函数默认仍检查探针 | schema diff 不能取代语义兼容测试；新增可选字段显式定义缺省，破坏改动需新版本 |

下阶段 M0-04 补完整提交响应、结果包 / 投影 / 签名、事务与独立控制契约；实际判定器和 18 / 19 YAML、受限控制写入、可靠检查点与 I4 网络实现随后接线。本轮不代表 H11 / M0 出口，也不解除 G 模块的来源许可 / 实时保证开放项。
