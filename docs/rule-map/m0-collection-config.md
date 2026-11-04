# M0-05B 规则与证据映射

这里记录已经执行的代码检查，不宣称整个 M0-05 或 M0 硬验收完成。

| 编号 | HLD 依据 | 实现 / 检查 |
| --- | --- | --- |
| B01 | 大纲 §4.4；分册一 §1 / §3 | manifests / collection_config：完整声明、显式池缺省、资源 / 能力收窄、Demo 单批与 checkpoint 禁用；配置测试含零值与空列表 |
| B02 | 大纲 §4.6、§4.11 | 来源绑定与控制保证门禁；原声明和覆盖都检查，late arrival 独立；非夹具 guarantee 和覆盖降级反例 |
| B03 | 大纲 §4.6；总册 §5 | 有效配置 / Schema / 决策文件锁、输入快照、周期与连续任务身份；重建校验与锁篡改反例 |
| B04 | 大纲 §4.3 | 不可变证据清单 / 六类写入方、实际字节校验、治理独立版本；角色全组合、同字节不同来源与父引用反例 |
| B05 | 大纲 §4.1 | 语义意图 / 外发记录分离，CE 顶层标量映射、父链、内联尺寸、source 引用、uint64；真实编解码与畸形 JSON 反例 |
| B06 | 大纲 §4.1 / §4.4 | 内建 registry 与具体 payload schema、等值路由过滤、属性分区；data 路径 / 无映射别名 / 非法排序拒绝 |
| B07 | 大纲 §4.6 / §4.7 | flags OR、manual_only 继承、Fake simulated 与 Null unsupported；输入与输出上下文篡改反例 |
| B08 | 大纲 §4.7；总册 §8.4 | prepare_bound_collection_material：独立证据根、PageResult / coverage / 事件 / 检查点的实际引用和内容摘要；重哈希伪造仍拒绝 |
| B09 | 大纲 §4.7 / §4.11 / §4.12 | violation 普通写入忽略但摘要保留；合法控制保留；已撤回内容要求真实剔除器，不能签普通输出 |
| B10 | 总册 §8.2–8.4 | make check / 专用测试 / 五组 Schema；远端 checks / infrastructure / isolation 和判定器全量附件按最终 PR head 验证 |

详细实际数量和 PR 证据见 [dev_memory](../../dev_memory/2026-11-04-m0-05b-collection-config.md)。真实鉴权、PG 创建与提交、I5 / I7 / I21、文件读取、I8 注册及外发尚未接入；Claim / 标准文档随 normalizer / Agent 落地，不能以本表替代这些验收。
