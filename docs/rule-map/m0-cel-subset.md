# M0-02D 规则—实现—验证映射

| 条款 | 实现 | 验证 / 边界 |
| --- | --- | --- |
| 大纲 §10：CEL 子集，不使用 Python eval | Rust CEL 绑定，内部 `evaluate_rule` | 布尔、比较、逻辑、算术、集合和宏；不承诺完整 CEL 一致性认证 |
| 分册一 §0.5.4：评估推荐库 | cel-python 类型反例后改选 common-expression-language 0.10.0 | ADR 0005；类型混用、列表比较、int64 溢出与精度回归 |
| 限制表达式长度 / 输入集合 | `_encode_request`，worker 复检 | 4096 字节表达式、128 项输入容器、1024 节点、16 层、64 KiB 请求，含 Unicode / 循环反例 |
| 限制可用函数 / 非确定性与任意扩展 | worker 编译后检查完整函数引用 | now / random / custom / timestamp / matches / 转换均拒绝；死分支也拒绝，注释 / 字符串不误报 |
| 控制求值成本或设超时 | 独立进程 + 有界 communicate + kill / 回收 | 真实导入 / 求值超时；高开销五层集合宏；PID 消失，后续正常；不宣称精确操作成本 / 内存上限 |
| 规则判断错误不可作为成功 | 精确 bool 返回与固定 RuleError.code | 语法、缺字段、类型、非布尔、超时、损坏协议、异常退出均失败，合法 false 正常返回 |
| 日志 / 追踪无来源正文 | 固定协议、禁库日志、父端丢弃 stderr | 真 worker stdout / stderr、错误消息不含合成来源标识 |
| 分册一 §8 / risk-grader：声明规则随执行锁版本化 | 留到 M1 正式领域包加载和 risk-grader | 当前没有规则表、Claim、证据或配置锁，不将原型当业务完成 |
| 用户：独立 PR、AI header、dev_memory | 独立功能分支、3 个新 Python 文件 header、详细记录 | 对应 memory 保存验证命令、选型与接手顺序；全部 CI 通过后合并 |
