# M0-02C：RFC 8785 / JCS 选型、向量与规范化原型

日期：2026-10-30（Asia/Shanghai）。工具：Codex(GPT)。分支：`feat/m0-jcs-canonicalization`。起点：PR #3 已合并，main=`028eabe22f0d66df02684320d97eec4958f0b8fb`。

## 1. 背景、依据与范围

用户要求继续下一项，依据 roadmap 进入 M0-02C。设计依据：大纲 §4.1 / §4.7；分册一 §0.4 的规范化与摘要、§0.5 第 3 项；分册二 §0.5 第 6 项。目标是实测候选、锁定库、以独立预期字节证明 RFC 行为，并为后续模型摘要提供内部原型。本阶段不定义正式跨组契约。

继续执行独立功能分支 → 本地验证 → dev_memory → PR → 实际 CI → merge；不向 main 直接提交。未委派子 Agent，仓库没有 `.codegraph/`，未自行建索引。应用 ponytail 原则复用 pytest、uv、make check 与既有 CI。新 Python 文件顶部包含 AI header / 工具 / 时间，新增函数均有 docstring。

HLD 的实现选型要求作为设计依据，与用户的执行授权区分；没有自行开展文档提及的额外人工审批或对外消息操作。新增库按总册 §11.1 二级实现变更信息在 ADR 0004 和 PR 记录动机、比较、影响、测试和回退，不修改冻结语义、跨组接口与 HLD 原件，不做 C 类裁剪。

## 2. 候选调查与实测

直接查阅 RFC 8785、Trail of Bits 实现源码 / 文档、PyPI 发布元数据和参考实现。2026-10-30 可安装的候选版本：`rfc8785 0.1.4` 与 `jcs 0.2.1`，均为 Apache-2.0、无运行时传递依赖。

使用隔离环境 `uv run --isolated --no-project --with jcs==0.2.1 --with rfc8785==0.1.4 python ...` 同时运行本仓库 26 个 Appendix B 位模式向量；逐行以 struct.unpack('>d', bytes.fromhex(bits))[0] 构造输入，对照非空预期 bytes；NaN / Infinity 要求抛错。两个候选均通过 26/26。

额外探针：对 `{1: 'test'}`，jcs 抛 AttributeError，rfc8785 抛 CanonicalizationError；对 `2**53+1`，jcs 接受并输出 9007199254740992，rfc8785 明确拒绝 IntegerDomainError。两个库都接受 tuple 为数组，因此平台边界仍需限制原生 JSON 类型。

选择 rfc8785：UTF-8 bytes API、明确的错误域和更保守的 Python 整数处理。只将 `rfc8785>=0.1.4,<0.2` 写入 pyproject，uv.lock 实际锁定 0.1.4；jcs 仅在隔离候选实验中安装，未加入应用或 CI。普通 json.dumps 不满足数字格式和 UTF-16 排序，不作 fallback。没有手写浮点转换算法，没有做完整 binary64 穷举或性能评测。

首次新增 pyproject 依赖后运行 make setup，按预期因 --locked 检测到锁文件未更新而退出；随后执行 uv lock，再 make setup 成功。最终锁文件检查通过，不修改 make setup 的 --locked 门禁。

## 3. 核心实现与数值边界

新增 `runtime/core/canonical_json.py`，为内部验证模块，尚未导出到 SDK / contracts / spi，也没有业务调用方：

1. `JSONValue` 描述原生 JSON 递归值。`_validate_value` 使用精确 type 检查，接受 null、bool、int、float、str、list、str 键 dict；拒绝 tuple、Decimal、模型、datetime、Enum、非字符串键和自定义子类。不调用对象的自定义序列化逻辑。
2. `canonicalize(value) -> bytes` 校验后交由 rfc8785.dumps。Python int 限 ±(2**53-1)；有限 float 按 binary64，负零输出 0。对循环 / 过深递归明确报错，不输出来源值。
3. `canonicalize_json(str | bytes) -> bytes` 仅接受 JSON 文本或 UTF-8 bytes。使用 json.loads 的 object_pairs_hook 在 dict 构造前拒绝重复键，parse_constant 拒绝 NaN / Infinity，parse_int / parse_float 都用 float 进入 RFC binary64 域；再经同一对象入口规范化。溢出到 Infinity、孤立代理项、BOM、非 UTF-8 和语法错误均失败。
4. `canonical_sha256(value) -> str` 返回 hashlib.sha256(canonicalize(value)).hexdigest()。不添加 URI 前缀、不补默认字段、不排除任何字段。
5. `CanonicalizationError(ValueError)` 对外使用固定消息并 `raise ... from None`；隐藏库异常可能包含的大整数、解析错误或来源键，不打印正文。

**必须保留的差异**：对象入口的 Python int 是精确整数，超安全域直接拒绝；文本入口按 RFC 解析 binary64，会将数字 9007199254740993 舍入为 9007199254740992。它不是无损契约解码器。64 位 ID / subjectseq 和高精度值按契约用字符串，先做模型校验与字段投影再调用对象入口；不能用文本入口绕过模型的整数规则。文本入口只输出 canonical bytes，不向调用方返回解析值。

Unicode 不做 NFC / NFD 归一化；字面量与转义后的相同键视为重复，但组合 / 分解形式仍不同。属性递归按 UTF-16 code units 排序，数组保持原次序，结果是无额外空白的 UTF-8。模型、时间、Enum 转换方式由后续契约显式确定，不能为通过本接口而随意 str()。

## 4. 向量、用例与核心文件

- `tests/fixtures/jcs/rfc8785-numbers.json`：RFC Appendix B Table 1 全 26 行（24 个有限数字 + NaN / Infinity 两个拒绝值）；第一列为大端 binary64 位模式，第二列为预期 ASCII 或 null。JSON 不加注释 header。通过独立脚本从下载的官方 RFC txt 表提取两列，与本文件逐项相等；不得由待测库重新生成预期。
- `tests/fixtures/jcs/README.md`：RFC 章节链接、归属、构造方式和离线说明；未复制参考实现代码。
- `tests/unit/test_canonical_json.py`：88 项用例。包括 26 行及每个有限值的文本再规范化、RFC 综合示例、非 BMP UTF-16 排序反例、6 种键排列 / 嵌套对象、摘要一致、数组和 null 差异、Unicode 形式保留、短控制转义、整数端点、非 JSON 对象、非法文本、重复键、非法编码、舍入与字符串身份、递归失败和错误脱敏。
- `Makefile` 新增 `test-jcs` 定向入口；原 make test / make check 自动收集全部非集成测试，无新增 CI job / marker / 在线数据下载。
- ADR：`docs/decisions/0004-jcs-canonicalization.md`；规则映射：`docs/rule-map/m0-jcs-canonicalization.md`。README、roadmap、memory 索引更新，并在 Kafka 原记录补 PR #3 最终合并证据。

## 5. 本地验证与 PR 证据

已实际执行：

- 候选实测：rfc8785 0.1.4 / jcs 0.2.1 各 26/26，通过并记录三种额外类型探针。
- 官方表比对：26 行与 RFC 下载原文逐项一致。
- `make test-jcs`：88 passed。
- `make check`：177 passed、29 个集成用例按 marker 排除；8 条 import-linter 契约保留、连接器导入、迁移链 1 revision 和 uv 锁文件检查均通过。总收集 206 项。
- 新增 2 个 Python 文件的 AI header、日期格式、函数 docstring 和 AST 语法检查通过；`git diff --check` 通过。

本轮不修改基础设施 / 数据库路径，未重复运行本地 Kafka 故障拓扑或 PG 集成；远端现有 CI 仍必须执行完整 PG / Kafka / observability 门禁。交付 [PR #4](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/4)，实现提交 `20dfb56`；最终 CI、合并状态和 merge SHA 以该 PR 的 GitHub 记录为准，全部检查通过后才合并。未执行的检查不记录为通过。

## 6. 复现与后续接手

```bash
rtk proxy make setup
rtk proxy make test-jcs
rtk proxy make check
```

如需复刻候选比较，使用第 2 节的隔离 uv 命令，将第 4 节 fixture 的两列逐行传给 jcs.canonicalize / rfc8785.dumps，按独立 expected 断言。仅安装选定依赖到应用。Python 仍为 3.12.12，运行测试不联网。

先读本记录和 ADR，再读核心模块、fixture 与测试、规则映射；下一项按 roadmap 是 M0-02D CEL 子集与成本 / 超时验证。后续契约阶段需：

1. M0-02E / M0-03 明确 Pydantic JSON 导出、枚举、时间、字符串数值域、schema / digest 元数据和模块归属；冻结后再决定是否提升为共享包，不能绕过 SDK 的既有导入边界。
2. M0-04 逐字段定义 semantic_bundle_digest include/exclude、缺省展开、稳定引用身份 + 版本 + 内容摘要；把完整包摘要与分段列表、策略版本、配置锁等绑定，再实现 HMAC / 回执验证。本原型的全值 hash 不替代这些逻辑。
3. 证据身份摘要对逻辑清单做 JCS；blob 字节摘要直接哈希原始 bytes，不把 blob 解码为 JSON 后哈希。
4. 接入不可信请求前，准入层补输入字节、节点 / 深度、执行成本或隔离上限。当前捕获 Python 递归错误不是完整资源预算，也没有独立超时机制。

没有业务 API、数据库表 / 迁移、事件 schema、身份或已有回执变化。当前可撤销模块 / 依赖 / 文档回退，无数据清理；一旦持久化摘要，升级必须审查身份与签名兼容性，不能更换实现后静默重算。RFC 向量测试不是全 M0 出口、语义摘要字段完备性或防伪签名验收。
