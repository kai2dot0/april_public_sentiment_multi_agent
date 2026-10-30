# ADR 0004：M0-02C RFC 8785 / JCS 选型与边界

日期：2026-10-30。依据：大纲 §4.1、§4.7；分册一 §0.4、§0.5 第 3 项；分册二 §0.5 第 6 项。选型结果为 `rfc8785 0.1.4`，通过 `uv.lock` 锁定。技术验证不表示语义结果包、证据身份或签名已经实现。

## 动机、候选与变更信息

同一逻辑输入必须产生同一串 UTF-8 字节，才能稳定计算摘要。Python 标准库 `json.dumps(sort_keys=True)` 的数字格式与属性排序不符合 JCS；不作为替代方案，也不在失败时回退。

2026-10-30 在隔离 uv 环境实测两个现成库，不引入自行维护的浮点格式化算法：

| 候选 | RFC Appendix B 26 行 | Python 非字符串键 | Python `2**53 + 1` | 选择 |
| --- | --- | --- | --- | --- |
| `rfc8785 0.1.4` | 26/26，含拒绝 NaN/Infinity | 明确 CanonicalizationError | 明确 IntegerDomainError | 使用；零运行时依赖，UTF-8 bytes API，错误域明确 |
| `jcs 0.2.1` | 26/26，含拒绝 NaN/Infinity | AttributeError | 舍入为 9007199254740992 | 仅作对照，未写入项目依赖与锁文件 |

两者均为 Apache-2.0，均接受 tuple 为数组；平台入口额外限制为原生 JSON 类型。没有依据本次向量测试推断全部浮点位模式已穷举或做性能排名。

按总册 §11.1，评估现成库属于已列出的验证路径；新增第三方库的影响按二级实现变更信息在本 ADR / PR 登记：动机和比较如上，影响为本轮内部原型及单元 CI，测试见下文，回退见末节。未修改 `contracts/`、`spi/`、锁序和大纲语义，无 A/B/C 裁剪；未把自动测试或 PR 合并记作额外的人工会签。

## 内部入口与数值域

`runtime/core/canonical_json.py` 为 M0 技术验证内部入口，尚未成为跨组冻结接口或 SDK 导出：

- `canonicalize(value) -> bytes`：只接受原生 null/bool/int/float/str/list/dict，键为原生 str；不隐式转换 tuple、Decimal、Enum、模型或 datetime。Python int 限 `[-(2**53-1), 2**53-1]`，float 按有限 binary64 处理。大整数不是先转 float 再接受。
- `canonicalize_json(str | bytes) -> bytes`：严格解析文本；bytes 只解码 UTF-8，拒绝 BOM、重复键（包括转义后重复和嵌套）、NaN/Infinity、溢出至 Infinity、孤立代理项和无效 JSON。所有数字词法通过 float 进入 RFC binary64 数值域，因此 `9007199254740993` 会按该域舍入为 `9007199254740992`。它是规范化工具，不是无损业务线解码器。
- `canonical_sha256(value) -> str`：SHA-256(JCS(value)) 的小写十六进制；覆盖传入的完整值，不自动插入或排除字段，不添加 `sha256:` / EvidenceRef 等业务包装。

两种入口的数值规则有意区分 Python 精确整数与 JSON 的 binary64 数字。精确 64 位 ID、计数器、subjectseq、大数或高精度业务值须由契约用字符串表示，先进行模型校验和语义投影，再规范化。不得用文本入口绕过已校验模型的安全整数规则；未来 Worker 线解码不可直接复用其解析值。

对象属性按未转义 UTF-16 code units 递归排序，数组保留次序。字符串不做 Unicode NFC/NFD 归一化。完整值的缺省、显式 null、字符串数字和数值各自保留语义。错误使用固定消息并抑制异常链，不记录来源内容；循环与过深递归明确失败。

## 验证与交付

`make test-jcs` 离线运行 88 项用例：RFC Appendix B 全 26 行（从十六进制 binary64 位模式构造）；§3.2.2 综合示例；§3.2.3 UTF-16 排序；6 种对象排列及嵌套；字节/摘要一致；数组与 null 差异；Unicode、控制转义、整数端点、非法类型、重复键、编码、异常脱敏及文本数值舍入。

26 行数据与下载的官方 RFC 表逐项比对一致，预期值不由待测库生成。其出处和归属见 `tests/fixtures/jcs/README.md`。正常 CI 的 `make check` 自动执行这些用例，不增加联网测试，也不依赖对照库。

当前无调用方、数据库或 API 迁移。后续 M0-02E / M0-03 / M0-04 确定模型导出、字段 digest 元数据与缺省展开，按字段生成明确投影后再计算 semantic_bundle_digest；原始 blob 摘要直接对字节计算，不经 JCS。HMAC、执行锁、证据清单、完整提交签名和回执验证仍待实现。

## 风险与回退

本轮不实现请求字节/节点预算、独立执行超时、模型语义校验或签名防伪。Python 递归上限的失败处理不等于完整资源隔离；正式接入不可信请求时应在准入层补充上限。文本舍入不能提供任意精度数值保真。

尚无业务数据依赖这些摘要，当前可通过撤销本模块、依赖和对应文档回退；不得回退为普通 JSON 序列化并继续签名。未来一旦进入持久身份与回执，库升级或摘要算法替换必须重跑向量与兼容验证，评估已有身份、签名与回放影响，不能静默重算。

来源：[RFC 8785](https://www.rfc-editor.org/rfc/rfc8785.html)、[rfc8785 源码与说明](https://github.com/trailofbits/rfc8785.py)、[rfc8785 0.1.4](https://pypi.org/project/rfc8785/0.1.4/)、[jcs 0.2.1](https://pypi.org/project/jcs/0.2.1/)。
