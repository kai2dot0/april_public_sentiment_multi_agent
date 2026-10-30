# M0-02E 工具链探针 schema

这里的三个 schema 是测试模型的导出快照，**不是已经冻结的 Worker / FetchPage 业务契约**。模型唯一来源为 `tests/support/contract_probe.py` 的 `PROBE_MODELS`；共享工具原型在 `contracts/tooling.py`。

- 线解码探针将枚举声明为字符串，保留未知值；构造侧对照模型使用 Literal。
- 每个声明字段带 `digest`、`default_semantics`、`routable`。元数据不等同摘要投影已实现，也不代表当前字段政策已经用于签名。
- 当前探针要求输入 / 输出 schema 同形，拒绝静默字段排除；普通检查只读，缺失、漂移、额外快照均失败。

```bash
make schema-check
make schema-write  # 修改模型后显式生成，审查 diff 并通过 PR
```

M0-03 / M0-04 才引入完整模型注册表、正式 schema 和兼容性测试；不要把测试 app 的 `/_probe/worker` 路径用于真实任务提交。Pydantic schema 差异门禁不能证明自定义 validator 的语义兼容性，后续仍须加场景测试。
