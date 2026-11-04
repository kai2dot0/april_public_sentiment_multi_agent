# 采集判定 v1 Schema

唯一来源是 `contracts/fetch_decisions.py` 的 DECISION_MODELS，两份根模型为 FetchDecision、DecisionPolicyLock。普通 `make schema-check` 只读比对；有意修改模型后执行 `make schema-write` 并审查 diff。

此前 v1 / v1-second 快照独立保留。规则文件自身通过封闭的 RuleDocument 严格加载，不是对外 Worker 载荷。完整调用语义见 [采集判定规范](../../../docs/contracts/fetch-decisions-v1.md)。
