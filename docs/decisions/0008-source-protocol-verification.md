# ADR 0008：来源协议核实与保守映射

核实时间：2026-10-30 23:54 至 2026-10-31 00:04（Asia/Shanghai）。依据：大纲 §8.3、§4.12；分册二 §0.5.2、§7、§8、§16；总册 §11.1 / §11.4。记录是技术调查结论，不代替来源资格、法务结论或 M0 出口。

## 选择与证据强度

后续开发选 **Jetstream v2 / `network.bsky.jetstream.subscribeEvents` / `xrpc.v1.json` / 实例内 seq 游标**，east 为主实例候选，west 只作需显式 handoff 的候选；v1 只保留兼容性与差异证据。实例完整 URL、源码提交、文件 SHA-256 在 [source-lock](../validation/m0-02g-source-lock.json)，公网实测在 [live-metadata](../validation/m0-02g-live-metadata.json)。两份记录都不是 ConnectorManifest，没有启用采集。

官方仓库现已分成 [Jetstream](https://github.com/bluesky-social/jetstream) 与 [jetstream-legacy](https://github.com/bluesky-social/jetstream-legacy)。新源码固定 `c9cbd1b3e6604b27bcb32cad08801b9806d4705a`，旧源码固定 `8a65de4eda28bed1cafcbcf25b0cd46ac6f2148b`；AppView / relay Lexicon 固定 atproto `a2dd244d6072216735198db5777909d014b0638e`；服务文档固定 bps-website `8a8a57d9df74b7a1eccbc58ac778609422da1e14`。**源码提交不是公网实例的部署版本**，后者 unknown。

当前[官方实例文档](https://bsky.network/docs/jetstream/)列出 v2 east / west 和 v1 east2 / west1；legacy README 的四实例列表是历史材料，不能直接当成当前 v2 清单。部署时必须重新核实具体实例。

## 协议映射

| 输入 / 行为 | 官方依据与本轮观察 | 平台保守映射 |
| --- | --- | --- |
| v2 数据帧 | 外层 `$type=message`，payload 的 `$type` 带 NSID + kind；事件 seq 位于 payload | 只提取外层 Jetstream seq；嵌套 account / identity / sync 的 relay.seq 不是本流游标 |
| v2 seq 恢复 | [固定 Lexicon](https://github.com/bluesky-social/jetstream/blob/c9cbd1b3e6604b27bcb32cad08801b9806d4705a/lexicons/network/bsky/jetstream/subscribeEvents.json)规定 inclusive / 至少一次 | 原样从最后持久位置恢复，允许重复；去重与数据库前沿稍后实现；不加一 |
| 过滤后序号不连续 | seq 属于完整实例事件流，过滤会跳过；重启也可能保留空位 | 不以 seq 差值判断本地丢项；本地接收顺序的持久连续前缀另算 |
| seq 低于 retention floor | 升级前 `HTTP 400 {error: CursorTooOld}`；east cursor=1 实测一致 | `cursor_expired`，停止，不静默转 live；后续 resync / replay 是独立流程 |
| timestamp 太旧 | v2 将 `cursor >= 10^15` 解释为 unix 微秒并夹到 floor，发无 seq 的 `#info OutdatedCursor`；west 实测一致 | `cursor_expired`，不接受夹断后的数据、不推进游标 |
| seq 超前 | `#info FutureCursor` 后上游可继续 live；east 实测提示映射 `invalid_job` | 停止，`invalid_job`；不能把上游仍可发送当成本次恢复成功 |
| terminal error | 外层 `$type=error`，不是 info；[固定 encoder](https://github.com/bluesky-social/jetstream/blob/c9cbd1b3e6604b27bcb32cad08801b9806d4705a/internal/subscribe/encoder.go)规定终止 | `transport_error`，没有可确认位置；未知帧 / 编码 / 枚举 fail closed |
| v1 | `kind / time_us / did` 外层，`wantedCollections / wantedDids`，无 v2 envelope；旧版未来时间直接 live | 不套 v2 解析器。v1 本次只验证 `requireHello=true` 暂停状态握手，不声称测过实际旧游标恢复 |
| sync | v2 有 `#sync`；当前 v2 encoder 注明 v1 wire 不发 sync | `sync_unverified`，不自动解除墓碑、不证明完整性；正式调度状态后续接线 |

v2 虽接受时间游标，本项目候选恢复格式只用 seq，禁止 seq / 时间数值域混用；时间只用于单独核实其夹断行为。事件 `time` 可受导入时间影响；`witnessedAt` 也只在单实例有意义，不产生跨实例复用资格。不要按人类可读时间或凭“差几秒”推导安全 handoff。

### 账户状态

[固定账户 Lexicon](https://github.com/bluesky-social/atproto/blob/a2dd244d6072216735198db5777909d014b0638e/lexicons/com/atproto/sync/subscribeRepos.json)将 `active` 定义为**发出事件主机上**是否可取 repo，不是全网永久存在性的布尔判定。实验分类：`active=false,status=deleted` 为永久删除候选；deactivated / suspended / takendown / desynchronized / throttled 为暂不可用；无 status、未知 status、矛盾组合均 unverified。`active=true` 不能证明内容级复活。真正墓碑、authority / generation、接收登记与 sync 处理留到正式控制面，探针不写任何状态。

### AppView

[getPosts Lexicon](https://github.com/bluesky-social/atproto/blob/a2dd244d6072216735198db5777909d014b0638e/packages/bsky/lexicons/app/bsky/feed/getPosts.json)规定最多 25 个 AT-URI。2026-10-30 23:59 实测 25 个虚构 URI 返回 200 / posts=[]，26 个返回 400。缺项以“请求集合减返回集合”计为 unresolved，不能确认删除；异常返回地址或重复地址必须拒绝。

[官方限流说明](https://bsky.network/docs/rate-limits/)确认 public.api.bsky.app 有缓存，但没有给出 AppView 固定数字额度或索引延迟 SLA；不能挪用 PDS 的 3000/5min。实测 200 的 `Cache-Control: public,max-age=30` 和 400 的 max-age=5 只是当时 HTTP 响应，Date / Age / TTL 都不能证明数据处理水位。删除确认需要的 freshness lower bound **unknown**；429 / Retry-After 需受控传输的预算退避，禁止靠压测寻找额度。

## 无证据的保证不启用

- retention 是实例配置，源码中的默认 / 上限不能当成公网 SLA，具体窗口 unknown。
- 本轮无真实接收事件对照，不能测得 live caught-up 标记、可靠上游接受时刻或控制事件有界投递保证。`control_delivery_bound=unknown`、`reception_time_proof_enabled=false`。
- 合成断线 / inclusive 重复验证的是客户端实验行为；实际公网的持久恢复、S2 资格、断线缺口、索引 freshness 与删除端到端均未验收。
- 缺少上述证据不阻塞契约和文件连接器地基开发，但真实来源不能据此取得首次资格或“缺项已删除”结论。

## 许可与用途登记

| 来源 | 能确认的公开依据 | 不能据此确认的用途 / 后续要求 |
| --- | --- | --- |
| Jetstream 软件 | [仓库 LICENSE-DUAL](https://github.com/bluesky-social/jetstream/blob/c9cbd1b3e6604b27bcb32cad08801b9806d4705a/LICENSE-DUAL)为 MIT / Apache-2.0 二选一 | 许可证针对软件，不自动覆盖帖子数据、模型处理或再分发；本轮没有复制上游实现 |
| Bluesky 公共网络 | 官方实例无需 live-tail 认证；[开发者指南](https://bsky.network/docs/developer-guidelines/)要求删除请求处理、安全与公开联系渠道；[TOS](https://bsky.social/about/support/tos)与[网络隐私通知](https://bsky.social/about/support/network-services-privacy-policy)可公开查阅 | 本次阅读没有建立本项目存储期限、向第三方模型传输、展示 / 摘录及再分发的完整用途依据；不虚构许可或负责人签字。真实接入前形成逐用途合规清单，落实删除时限、责任人和再核实日期 |
| HLD 所举国家统计局 RSS | [服务条款](https://www.stats.gov.cn/wzgl/202302/t20230217_1912857.html)要求注明来源，并排除第三方、禁止转载、需特别授权等内容；本次两个 feed HEAD 均 200 / text/xml，robots GET 404 | robots 404 只是本次没有可用规则，不代表用途授权；feed 可访问不代表第三方全文可用。逐文章核对归属、引用和模型用途；RSS 解析 / 删更语义尚未开发 |

TOS / 隐私通知经网页工具阅读，直接 HTTP 取 TOS 遇到 403，因此没有伪造其字节摘要；source-lock 单列 web_review_only。没有向站点发消息、申请权限、登录、抓取真实帖子 / feed 正文、发送第三方模型或建立私有镜像。目标环境继续遵守 HLD 的切片 / 墓碑导入边界。

## 实现与变更影响

新代码只在 `tests/support/source_protocol_probe.py`、对应测试与 `scripts/probe_source_metadata.py`。复用 WireModel / decode_wire 的严格 JSON 边界及 AckPrefix；没有生产业务解析器、正式 Manifest、数据库迁移、contracts / spi 修改、事务或锁序变化。

新增开发依赖 `websockets==17.4`（声明 >=17.4,<18，uv.lock 固定），用于真实 loopback WS 与显式手动元数据探针，BSD-3-Clause；不手写 WS framing，不放进 connector、不提供任意 URL 的业务网络出口。按总册 §11.1 记录为二级组件选型信息；无 A/B/C 裁剪、无虚构跨组签字。具体疑问见 [核实登记表](../rule-map/m0-source-protocol.md)。

`make test-source-protocol` 离线，纳入默认 check；`make probe-source-metadata` 显式访问四次固定 WS / 两次 HTTP、虚构 DID / URI，只输出端点、状态、已知错误、提示分类、缓存头与计数，无原始正文 / 作者 / 错误 message。握手、接收、关闭均有时间预算，不自动重试。公网瞬时失败如实记录，不成为默认 CI 的不稳定依赖。正式受控网络和预算服务还未接入，不能把该脚本当作生产采集入口。

回退只需移除测试探针、Make 入口和开发依赖；无数据迁移。后续实现必须维持停止映射及 unknown 边界，不能为通过演示把上游 silent clamp 当作成功。
