# M0-02G 规则映射与技术核实登记

此表保留总册 §11.4 的字段语义，仓库是本轮可核对载体；没有访问或声称同步 Confluence，也没有替任何指定负责人签字。提出人为 Codex，2026-10-30 / 31。答复列为技术调查，不是文档中指定角色的正式审批。

| 编号 | 文档 / 章节 | 疑问与建议 | 影响范围 | 答复 / 状态 | 结论依据 | 变更级别 | 落实 / 验证 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| G-01 | 分册二 §0.5.2 / §7、大纲 §8.3 | v1 / v2 名称和实例有变化，不能共用帧与游标；选 v2 | 二组；未来 I4 | 调查已落实；正式连接器待接入 | 固定 Lexicon / encoder / 官方实例页 | 二级组件实施记录；无契约改动 | source-lock；v1 / v2 隔离、实例绑定、数值域反例 |
| G-02 | 大纲 §8.3 | CursorTooOld 不是普通 advisory，而是升级前 400；时间过旧有无 seq 的 OutdatedCursor | 大纲映射表述 / 二组 | 调查已落实，保守停止语义不变；待正式角色确认文档表述 | 固定 cursor.go / Lexicon；公网两种过旧结果 | 兼容性澄清；不放宽大纲底线 | HTTP / frame 分类分离，提示不推进位置 |
| G-03 | 分册二 §7 | FutureCursor 后服务器可继续 live，是否仍成功 | 二组 / 运行时 | 调查已落实，平台停止为 invalid_job | 公网 future seq；固定 Lexicon | 无语义变更 | 真实 loopback 提示后还有数据，但客户端不消费为恢复结果 |
| G-04 | 分册二 §0.5.2 / §7 | active=false 与 deleted 混同会误删 | 二组 / 删除控制面 | 调查已落实，正式权威状态待实现 | relay Lexicon 的 host 语义和 knownValues | 无规则裁剪 | 10 种状态组合，deleted 也只形成候选 |
| G-05 | 大纲 §8.3 | 25 URI、缺项、缓存 TTL 能否证明删除 | 二组 / 证据服务 | 上限核实完成；新鲜度能力保持 unknown | getPosts Lexicon、25/26 公网结果、rate-limits | 无语义变更 | 缺项 unresolved；Date / Age / max-age 均不建立索引下界 |
| G-06 | 分册二 §0.5.2 | 是否有有界 control 投递和 caught-up 证明 | 跨组 / 首次资格 | 未找到保证，保持开放；真实接入前核实 | 当前公开文档不提供所需保证；本轮无公网事件对照 | 无变更，禁用接收时间证明 | source-lock 明确 unknown / false，回归保护 |
| G-07 | 大纲 §8.3、分册二 §16 | 软件开源 / API 公开是否等于内容和模型用途许可 | 跨组 / 语料 / 网关 | 公开材料调查完成；具体用途资格仍开放 | LICENSE-DUAL、开发者指南、TOS、隐私通知 | 来源启用前置；不自行宣布批准 | collection=false；按存储 / 模型 / 展示 / 删除期限补用途清单 |
| G-08 | 分册二 §8 / §16 | RSS 示例是否能直接转载全文 | 二组 / 中文语料 | 示例端点与条款初核；接入前逐篇复核 | stats.gov.cn 两个 feed HEAD 200；robots 404；服务条款有例外 | 无范围扩张 | 没有 feed 正文入库；RSS 解析 / 许可链待后续 |

## 规则—实现—验证

| 条款 | 交付 | 边界 |
| --- | --- | --- |
| 分册二 §0.5.2：版本、实例、帧与错误核实 | 10 个固定来源摘要 + 网页政策依据；四次 WS / 两次 HTTP 元数据 | source commit 不冒充部署 commit；v1 只做暂停流握手 |
| seq / 时间 / relay 位置分开 | ProbeCursor、v1 / v2 各自外层、数值域检查 | 不是正式 Cursor / Manifest 契约 |
| inclusive / 连续前缀 | 真实 loopback 断线、11/99、从持久 11 重复恢复，复用 AckPrefix | 无数据库持久事务；filtered gap 不按数值补齐 |
| advisory 不前进、异常明确失败 | 过旧 / 超前 / 未知 info、terminal / HTTP 分离、严格 decode_wire | 64 KiB 是合成实验预算，未定义生产帧上限；不作完整 Lexicon 校验 |
| account / sync 保守 | deleted 候选、临时不可用、unknown / sync_unverified | 无墓碑写入、复活、authority / generation 接线 |
| AppView 上限 / 缺项 / freshness | 地址集合校验、25 / 26、缓存头反例 | 无 rate-limit 压测、没有确认真实帖子删除 |
| 用户要求 | 独立分支 / PR、三个新 Python 的 AI header / 函数说明、详细 memory | 三项 CI 全成功再合并；不把本轮当作完整 M0 或来源合规已批准 |
