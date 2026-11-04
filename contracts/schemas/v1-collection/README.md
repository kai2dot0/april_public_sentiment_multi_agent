# 首个采集闭环 Schema v1

由 `contracts.collection_registry.COLLECTION_MODELS` 导出的 16 个根模型；复用模型作为各根的 `$defs`。包含 Manifest / 部署缺省 / 来源 / 有效配置、任务 / 身份 / 输入 / 文件查询、事件注册 / 描述 / 意图 / 外发记录 / 元数据、证据清单 / 治理与依赖 flags。

`make schema-check` 只读校验漂移；修改模型后显式 `make schema-write` 并审查生成差异。EventRecord 是内部记录 Schema，CloudEvents 线格式经 `sdk.events` 显式转换，不能直接输出 routing / content_kind 存储字段。新增 profile 不改变此前 29 + 41 + 2 个正式快照。

调用与信任边界见 [消费规范](../../../docs/contracts/collection-config-v1.md)，设计与版本影响见 [ADR 0012](../../../docs/decisions/0012-collection-config.md)。
