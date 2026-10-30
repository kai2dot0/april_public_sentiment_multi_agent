# 开发阶段与交付顺序

以 HLD 总册 §3、§7.3、§8、§9 为依据。编号是工程切分，不能替代 HLD 里程碑；每个交付单元均独立 PR、验证和维护 `dev_memory`。阶段内按冻结接口并行、尽早集成；当前没有委派其他 Agent。

| 单元 | 范围 | 验收 / 状态 |
| --- | --- | --- |
| M0-01 | 工程目录、依赖锁、API 存活、基础设施、迁移链与基础 CI | 已通过 [PR #1](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/1) 合并；不表示 M0 出口 |
| M0-02 | 第 1 周技术验证：PG 锁/守卫/屏障、Kafka 幂等与 ACK、JCS、CEL、进程隔离原型、协议核实 | 按下表独立交付；PG / Kafka / JCS / CEL / 契约工具链已合并，本轮实现隔离验证，来源协议仍待核实 |
| M0-03 | 第 1 批契约：执行上下文、传输、FetchPage/条目结果、错误、ID 派生输入 | 待冻结并实现；双方接口一致 |
| M0-04 | 第 2 批契约：结果包/摘要、完整提交、事务上下文、控制对象与锁、状态服务 | 待冻结并实现；事务与权限语义明确 |
| M0-05 | 最小运行时、SDK、文件连接器、证据 API、I5/I7/I21 最小真实适配器、注册 Fake | 待开发；总册 §8.2 集成点 1：文件一页 → 提交 → 检查点 → 事件；不依赖 I9 |
| M0-06 | 真实注册、撤回、派生写入屏障、I9 最小真实、定时对账、进程与网络控制、异常夹具 | 待开发；完整覆盖 M0 H1–H17 及子项，硬判据不带条件通过 |
| M1 | 真实来源、网关、嵌入、知识/指标、分析 Agent、领域包、报告、评测与列表 | 待开发；真实增量与离线语料、删除全链路、报告两级就绪 |
| M2 | 回放/升级、第二包、resync 清扫、故障/容量、证据钻取与漏报评测 | 待开发；完成组合评估与加固，不沿用不同模型的旧评测 |
| M3 | 目标环境切片/墓碑导入、死信、回归、内部/外部演示与彩排 | 待开发；按总册出口逐项验收 |

## M0-02 分项

各分项独立 PR 和 memory；技术验证不等于业务能力已实现，不将部分通过标为 M0-02 整体完成。

| 分项 | 交付 | 状态 |
| --- | --- | --- |
| M0-02A | PG 行锁 16 对、SKIP LOCKED、SET LOCAL、Read Committed、死锁/整事务重试、守卫与命名屏障 | 已通过 [PR #2](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/2) 合并 |
| M0-02B | Kafka 幂等生产者配置、ACK、重试与确认行为 | 已通过 [PR #3](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/3) 合并 |
| M0-02C | RFC 8785/JCS 候选与测试向量 | 已通过 [PR #4](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/4) 合并 |
| M0-02D | CEL 子集与成本/超时约束 | 已通过 [PR #5](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/5) 合并 |
| M0-02E | Pydantic/schema/digest 元数据、未知枚举、Worker 传输形态、穷举规模验证 | 已通过 [PR #6](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/6) 合并；正式模型、完整 H11 仍待后续阶段 |
| M0-02F | 进程组、cgroup、网络与缓存清理原型 | 本轮实现；见对应 dev_memory、ADR 0007 与专用 isolation CI |
| M0-02G | Jetstream/AppView 协议、游标、缓存与许可核实 | 待核实，不以设计文字代替实际来源证据 |

## 当前已知约束

- M0-01 中新增 uv 作为工程工具，选型与边界见 ADR 0001；其他业务技术候选不在本 PR 中决定。
- 校验库骨架归一组；M1 领域包加载/锁定/注入已转交一组；第二夹具包仍归二组。
- 生成类模型先接已登记方案 DeepSeek，资源/合规须落实；嵌入在平台内。模型网关尚未实现，本阶段无模型密钥与外部采集。
- 一期数据源边界、假名化、删除义务均按 HLD，文件夹具不代替真实业务组合评测。
- 当前 CI 仅覆盖已开发骨架。每次新增接口、存储、迁移和工作流都需补对应门禁；迁移升级后要调整测试中的期望版本。
