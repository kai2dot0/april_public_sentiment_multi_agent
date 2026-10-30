# 开发记录索引

每个功能、模块或重要设计建立新的记录；同一功能的修正、验证和后续扩展继续更新原文件。记录必须足以让其他 Agent 接手，并区分设计目标、已实现内容和实际验证结果。

| 记录 | 范围 |
| --- | --- |
| [2026-10-30-m0-01-project-foundation.md](2026-10-30-m0-01-project-foundation.md) | 首个工程骨架、开发环境、CI、迁移与接手顺序 |
| [2026-10-30-m0-02a-postgres-concurrency.md](2026-10-30-m0-02a-postgres-concurrency.md) | PG 锁序守卫、命名屏障、真实锁矩阵、死锁与重试实验 |
| [2026-10-30-m0-02b-kafka-delivery.md](2026-10-30-m0-02b-kafka-delivery.md) | Kafka 幂等配置、ACK 前沿、真实 ACK 丢失/超时与重发实验 |
| [2026-10-30-m0-02c-jcs-canonicalization.md](2026-10-30-m0-02c-jcs-canonicalization.md) | JCS 候选选型、RFC 全部数字向量、严格 JSON 边界、摘要原型与后续接线 |
| [2026-10-30-m0-02d-cel-subset.md](2026-10-30-m0-02d-cel-subset.md) | CEL 选型反例、函数子集、输入规模、真实超时回收与后续领域包接入 |
| [2026-10-30-m0-02e-contract-toolchain.md](2026-10-30-m0-02e-contract-toolchain.md) | schema / 字段元数据、开放枚举、真实 HTTP/JSON、有限维度扫描与契约冻结接手 |
| [2026-10-30-m0-02f-worker-isolation.md](2026-10-30-m0-02f-worker-isolation.md) | cgroup 替代方案、四角色内核网络、OOM / 整树终止、缓存清理与控制面接手 |
| [2026-10-30-m0-02g-source-protocol.md](2026-10-30-m0-02g-source-protocol.md) | Jetstream 两代协议、实例游标、真实 WS 恢复、AppView 缓存与许可证据、未知保证及后续接线 |
