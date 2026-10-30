# JCS 数字测试数据

`rfc8785-numbers.json` 逐项转录 [RFC 8785 Appendix B Table 1](https://www.rfc-editor.org/rfc/rfc8785.html#name-number-serialization-sample) 的 26 行：第一列是 binary64 大端位模式，第二列是预期 ASCII 输出；NaN / Infinity 的空输出记为 `null`，测试必须拒绝。

来源：RFC 8785，A. Rundgren、B. Jordan、S. Erdtman，June 2020，Copyright © 2020 IETF Trust and the persons identified as the document authors。这里只保存数字向量数据，未复制参考实现代码。输入不能先转十进制 float 字面量，否则可能掩盖边界位模式；测试通过 `struct.unpack` 从十六进制构造。

测试中的 §3.2.2 综合示例和 §3.2.3 排序示例也以 RFC 的独立预期值断言，预期结果不由待测库生成。测试运行无需联网。
