# 采集判定规则

18 行动作、19 条违规按大纲 §4.11 编号，唯一可执行来源是本目录 YAML。文件使用 YAML 1.2 JSON 流式语法及顶部注释，加载器只支持该封闭子集；任何修改均改变字节摘要。

规则文件、evaluator_version 与两张表的版本共同钉定；变更正式枚举也要同步更新表 / 清单并跑 `make test-fetch-decision-matrix`。一般 `make check` 含页面反例；CI 每 PR 额外完整扫描并保存逐违规组合附件。

格式、权威输入、签名前接线、预算 / 提交边界和 artifact 重放见 [消费规范](../../docs/contracts/fetch-decisions-v1.md)。
