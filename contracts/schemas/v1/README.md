# 第一批正式契约 v1

唯一字段来源是 `contracts/registry.py` 中注册的 Pydantic 模型；此目录为 JSON Schema 2020-12 快照。`make schema-check` 只读比较，`make schema-write` 显式生成后必须审查 diff；探针快照仍在 `../probes/`，不能替代本目录。

所有声明字段有 `digest`、`default_semantics`、`routable` 元数据。第一批均 include，**未实现语义包投影 / 签名**；动态检查点、来源覆盖详情和未验证控制载荷也必须是有限 JSON，宽整数编码为字符串。Schema 不完整表达 Python 的交叉字段 / 日历校验，兼容测试和模型校验仍须运行。

- `fetch-page-wire` 是不可信线输入，枚举字段保留原始字符串；`fetch-page` 是构造侧已知枚举，不能拿它接收 Worker 请求。
- FetchPage 控制载荷暂以 JSON 对象保留；第二批控制契约将单独验证、鉴权和投影。可解码不表示可写入，未知状态不丢弃墓碑。
- `record-receipt-command` 是连接器接口；`trusted-receipt-registration` 仅可信 SDK → 传输服务。后者不得直接暴露给插件。
- 对不可信字节使用 `contracts.tooling.decode_wire`，不能以 BaseModel 原生 JSON 解析绕开重复键 / 非有限数值 / UTF-8 检查。
- 当前是工程实现基线，未伪造 HLD 指定的接口负责人签字；完整提交协议、结果包、事务上下文和控制状态服务在 M0-04。

规范与边界见 `docs/decisions/0009-first-contracts.md`、`docs/contracts/first-batch-v1.md` 和本模块 dev_memory。
