# ADR 0005：M0-02D CEL 函数子集与执行超时

日期：2026-10-30。依据：大纲 §10（YAML 决策表 + CEL 子集）；分册一 §0.4、§0.5 第 4 项、§8 的声明式领域包和 risk-grader。选择 `common-expression-language 0.10.0`（Rust CEL Python 绑定），以 uv.lock 固定依赖。本轮为内部技术验证，无业务规则表、Claim 或领域包加载接口。

## 候选与变更说明

先实测 HLD 推荐评估候选 `cel-python 0.5.0`，发现其 BoolType 继承 Python int，原生求值会接受 `true + 1 == 2`、`true == 1`、`[true] == [1]`，并接受 int/double 混算。这与目标 CEL 类型语义不一致；不为它手写一套运算符和类型修补。

改用 `common-expression-language 0.10.0`：实测布尔/整数比较为 false、混合算术抛类型错误、int64 溢出报错，支持完整函数引用静态检查。原 cel-python、Lark、RE2、Pendulum 等候选依赖均已从项目锁文件移除。选定库 Apache-2.0，原生 wheel 在 macOS ARM64 本机安装并验证，Linux x86_64 由本 PR CI 验证；不承诺尚未测试的部署架构。

此为总册 §11.1 二级实现变更（替换推荐候选且新增组件）：动机和方案比较如上；影响仅内部 runtime 模块、依赖锁与单元 CI；验证和回退如下。保留大纲 CEL 子集，不引入 Python eval、其他规则语言或跨组契约变更，无 A/B/C 裁剪。PR 记录实现评审，不虚构文档中其他人工角色已会签。

库自带 CLI 依赖 Typer、Rich、prompt-toolkit、Pygments；当前仅使用 Python 编译 / 执行 API，不启动 CLI，不加载可选 stdlib 扩展。锁文件净新增该库及 prompt-toolkit、shellingham、typer、wcwidth 五个包，Rich / Pygments 已在原依赖树中。

## 子集与错误语义

内部 API：`runtime/core/cel_rules.py:evaluate_rule(expression, facts, *, timeout_seconds=2.0) -> bool`。

- 唯一全局输入为原生 JSON dict `facts`；精确 int64、有限 double、bool、str、null、嵌套 list / str 键 dict。不同于 JCS 文本解析，本协议保留 Python / JSON int64 精度，不把数字先转 float。禁止 datetime、callable、模型实例及隐式对象转换。
- 保留核心 CEL 表达式、字面量、比较、算术、逻辑、条件、索引与集合宏（has / all / exists / exists_one / map / filter，含嵌套）。最终结果必须是原生 bool，非布尔值报错；不以 Python truthiness 判定。
- 编译后、执行前，通过固定版本的 `Program.functions()` 检查完整引用集合，只允许核心运算符、宏展开的 `@in` / `@not_strictly_false` 和 `size` / `contains` / `startsWith` / `endsWith`。命名函数白名单不可由调用方扩展；不注册 Context 自定义函数、Python 回调或 stdlib 扩展。
- timestamp / duration、数值 / 字符串类型转换、type、matches、now、random、自定义函数均拒绝，包括死分支内的调用。宏展开后是核心运算，不用函数名扫描或正则匹配 CEL 源码；字符串和注释中的函数名不误报。未绑定变量在实际求值时按 CEL 返回错误，未注册消息构造不可创建宿主对象。
- 错误分为 invalid_input / input_limit / syntax / unsupported / evaluation / non_boolean / timeout / worker_failure；任何错误都不伪装为合法 false。

## 规模、超时与进程生命周期

| 限制 | 当前固定值 |
| --- | --- |
| 表达式 UTF-8 | 4096 字节 |
| 整个 stdin JSON 请求 | 65536 字节 |
| 每个输入 list / dict | 128 项 |
| 输入总节点（包含对象键） | 1024 |
| 输入嵌套深度 | 16 |
| 单个输入字符串 / 键 UTF-8 | 4096 字节 |
| worker communicate 截止时间 | 缺省 2 秒；调用范围 `(0, 5]` 秒 |

准入先检查规模 / 原生类型，再编码；不对任意大容器先逐项扫描。循环结构由深度 / 节点限制拒绝。子进程独立重复验证字节和输入限制。

每次求值使用当前锁定 Python 启动 `-m runtime.core.cel_worker`，固定项目工作目录，表达式和输入仅经 stdin，不进入命令行。截止时间覆盖 worker 导入、编译、引用检查与求值；不声称操作系统创建进程本身可被该计时器中断。超时或父调用中断时执行 kill + communicate 回收进程；不留下后台求值线程。

worker stdout 仅含布尔结果或固定错误码，禁用库日志。父进程丢弃 stderr，不把解析异常、激活变量、来源正文或命令输出拼入错误；异常退出和损坏协议均报 worker_failure。仅允许协议规定的键和精确 bool 类型。

本轮选择 HLD 允许的“设超时”路径，**不提供精确 CEL 操作数计量、AST 节点成本、派生中间集合上限或硬内存上限**。嵌套宏可能达到乘法级工作量，由截止时间终止；超时不能保证所有平台内存峰值。独立进程不是完整安全沙箱，尚无进程组 / cgroup、凭据剥离、文件系统或默认拒绝网络控制，这些属于 M0-02F / M0-06。没有可供表达式使用的宿主 I/O 函数，但不以此替代 OS 隔离。

## 验证、后续与回退

`make test-cel` 的 111 项用例覆盖正常与 false 判断、核心宏、CEL 负数整除、int64 精度、类型反例、完整函数引用、输入限额、非布尔拒绝、真实启动超时与 128 元素五层 all 高开销超时、kill 后 PID 消失、后续求值正常、协议损坏和输出脱敏。完整 `make check` 自动包含这些离线测试，CI 继续跑原有 PG / Kafka / 可观测检查。

无数据库迁移、冻结接口、事件 schema 或持久化规则结果变化。M0-02E / 后续契约阶段确定参数和错误封装；M1 领域包加载与 risk-grader 接入时，将库版本、规则版本和预算锁进执行配置并补场景评测。不能以本原型代替 YAML 决策表全组合穷举、规则证据或 decision Claim 提交。

当前可撤销模块 / 依赖 / 测试回退，无业务数据清理；不得自动回退到不合格候选或 Python eval。未来规则结果一旦进入审计与回放，升级引擎需重跑语义反例及业务规则评测，评估结果变化，不能静默更换。

来源：[CEL 语言定义](https://github.com/cel-expr/cel-spec/blob/master/doc/langdef.md)、[HLD 候选 cel-python](https://github.com/cloud-custodian/cel-python)、[选定绑定源码](https://github.com/hardbyte/python-common-expression-language)、[0.10.0 发布](https://pypi.org/project/common-expression-language/0.10.0/)。
