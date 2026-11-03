# 第二批 v1 Schema

唯一来源为 `contracts/second_registry.py` 的 41 个公开根模型；共享嵌套模型由 `$defs` 导出。每个字段声明 digest、default_semantics 与 routable；模型严格拒绝额外字段 / 类型转换，Schema 之外的跨字段约束仍须服务端模型校验。

`make schema-check` 只读比较 probes / v1 / v1-second 三组快照；模型有意改变后才执行 `make schema-write` 并审查 diff。JSON 不添加代码 header。这里不是部署 API 或组织签字记录。

消费约定、签名、回执优先级、事务和限制见 [第二批规范](../../../docs/contracts/second-batch-v1.md)、[ADR 0010](../../../docs/decisions/0010-second-contracts.md)。第一批快照独立在 `../v1/`，不能从历史 probes 复制业务定义。
