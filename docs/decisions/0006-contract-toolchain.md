# ADR 0006：M0-02E 契约工具链、Worker 传输与组合规模

日期：2026-10-30。依据：分册一 §0.5 第 5–7 项、§1、§3、§6；大纲 §4.1、§4.7、§4.9、§4.11；总册 §3、§8.4、H11。本轮复用现有依赖，尚未冻结正式 FetchPage / Worker 业务模型。

## 工具链与影响级别

选定既有 Pydantic v2 → JSON Schema 2020-12 路径。锁定环境实际为 Pydantic 2.14.2、FastAPI 0.162.0、httpx 0.28.1、uvicorn 0.54.0；无依赖变更。

新增 `contracts/tooling.py` 作为独立叶子工具原型：WireModel、字段元数据、检查后导出 schema、严格 JSON 解码。测试模型唯一来源为 `tests/support/contract_probe.py`，快照仅位于 `contracts/schemas/probes/`，明确不代表业务契约冻结。SDK / runtime 没有接入这些探针。

HTTP/JSON 是 HLD 推荐内的一级选型；新增共享 contracts 工具按总册 §11.1 的三级影响信息记录：没有已存在的跨组业务调用方，返工 / 迁移量为零，不更改业务字段、事务或锁序。正式两批接口及签署状态仍为待冻结；PR 合并不虚构额外人工会签。无 A/B/C 裁剪，没有修改 HLD 原件或大纲语义。

## 模型、元数据和 schema 门禁

- WireModel 使用 strict、extra=forbid、validate_default、allow_inf_nan=false、hide_input_in_errors。未知**字段**明确拒绝；未知**枚举值**由接收模型的 str 字段原样保留，二者不同。
- `field_metadata` 集中声明 `digest: include|exclude`、`default_semantics`、`routable`。`checked_schema` 检查每个嵌套声明属性，拒绝子模型额外字段，要求本探针输入 / 输出 schema 同形，防止静默字段排除。字段元数据不是字段投影实现，也不是对任意 exclude 字段的业务授权。
- 三个注册 schema：page-wire-probe、strict-status-probe、worker-envelope-probe。开放线模型使用 str；严格构造对照使用 Literal。CI 从同一注册表只读比对快照集合及内容；缺失、漂移和未登记 schema 都失败。显式 `make schema-write` 后必须审查 diff。
- 默认值明确展开为 null / 空数组；它们写在模型与元数据中。默认展开后的值参与往返一致性实验，不声称稀疏原始 JSON 与展开后的 JSON 字节相同。
- 严格字节入口拒绝重复键（含转义后重名）、非 UTF-8 / BOM、非法 Unicode、NaN / Infinity、数值溢出及结构损坏，固定错误且抑制异常链。不用会舍入的 JCS 文本解析器充当业务线解码器。

schema 差异检查不是完整兼容性证明：自定义 validator 语义、默认含义和未来字段投影仍需场景测试与评审。当前严格配置和同形限制是探针选择，不替代正式版本演进政策。

## 未知枚举路径与数值发现

测试页片段包括 status、completeness、end_reason、failure_scope、status_reason、item_outcomes.outcome 和合成墓碑。所有枚举位置接收原始 str，不 trim / 映射 unknown；status_reason 只在 forbidden 下检查枚举，其他状态下是自由诊断文本。

`unknown_enum_paths` 只验证 V1 的识别路径。未知 status 的页在真实 TCP 往返后仍保留墓碑和条目；损坏结构进入固定 422。**没有完整 V1–V19 verdict、控制权限验证、受限提交投影或数据库墓碑登记**；这些仍按 M0-03 / M0-05 / M0-06 接线。

int64 在 Python / HTTP JSON 中可精确保留，不代表可进入 JCS 对象入口的安全整数域。探针故意验证 int64 最大值解码保真且 JCS 拒绝；subjectseq 按 HLD 用字符串保留完整 64 位值。M0-03 须明确代次 / ID 的安全数值范围或字符串编码；不能用 float 转换规避拒绝。当前没有真实 lease 服务使用探针字段。

## Worker 形态结论

采用 HTTP/JSON。已有 FastAPI / Pydantic / httpx 可直接复用，JSON 线格式易检查；不增加 gRPC / protobuf / schema registry。没有做 gRPC 吞吐对比，也不把本次往返当作容量结论。

测试从已有 create_app 工厂扩展独立探针 app，以随机 loopback 端口运行真实 uvicorn；启动事件同步、客户端禁环境代理、服务线程有界关闭。测试路由 `/_probe/worker` 只做 codec echo，**没有挂到生产 create_app**，其 200 不代表任务提交成功。正常模型路由和 OpenAPI 仍来自 Pydantic；测试中间层先检查严格字节解码，避免框架默认 JSON 覆盖重复键。探针请求阈值 64 KiB 在读取后检查，仅为测试阈值，不宣称流式内存准入已实现。

认证身份、租约与 fencing、错误到业务结局映射、超时重试幂等、请求资源限制和完整端点在后续正式协议阶段实现。当前只有传输技术结论。

## bounded_slice 维度规模与 CI 策略

完整维度数据及实测见 `docs/validation/m0-02e-matrix-scan.json`。状态等已知域从探针 Literal 声明取得，每个枚举位置加入未知代表值；status_reason 另有自由文本。item_outcomes 按 HLD 五类聚合，再显式扩展含未知 outcome 的四类（集合内 / 外 × 含 / 不含 missing），共九类。它是有限代表值轮廓，不是对任意页内容和所有诊断字符串的穷举。

调度 / 请求方式按已校验的平台上下文取已知值，未知代表值覆盖 V1 列出的页面枚举。正式判定器接入前须复核聚合类，尤其 V8 对查询页 failed / missing 和观测集合的区分；若需细分或上下文域变化，必须重算，不能沿用本轮总数作为最终 H11 固定规模。

维度大小：`2×12×5×4×4×6×2×2×2×9×2×2 = 3,317,760`。仅排除 HLD 明示的“single_resource × query_or_stream”829,440 个上下文，不赋 verdict；剩余扫描 2,488,320。没有将语义违规组合当作不可达排除。

显式 itertools.product 全扫，未知枚举探针命中 1,915,920；这不是完整违规组合数。分片按原始笛卡尔序号 `ordinal % shard_count`，三片和的计数与 checksum 等于全量；JCS profile_digest 绑定**有序**维度名称 / 取值数组，避免维度重排后沿用旧分片证据。

2026-10-30 22:48 本机 Darwin ARM64 / Python 3.12.12 全扫描约 **0.409 秒**，额外内存不保存全部组合。测量只含组合扫描和未知值探针，不含完整 19 条校验、18 行动作匹配或事务，因而仅是耗时下界。CI 每个 PR 全扫并打印独立测量；当前规模无需夜间专用扫描或强制分片。

正式判定器接入后必须重测：决策表 / 违规清单 / 判定器变化跑全量，其他 PR 可确定性分片、夜间全量；以实际耗时再决定并行度。**本记录不能作为 H11 通过证据**，不能据 0.409 秒承诺完整决策表运行时长。

## 验证与回退

新增 52 项测试，完整 make check 为 340 项通过，包含 8 条架构契约、schema、迁移链和锁文件检查。真实 HTTP 验证未知枚举 / 墓碑保留、编码及重复键拒绝、OpenAPI 和生产路由分离；测试正常 / 漂移 / 缺失 / 额外 schema、嵌套元数据、默认展开和分片计数。远端仍跑完整 PG / Kafka / 可观测门禁，并输出 Linux 扫描测量。

没有依赖升级、数据库迁移或持久业务数据；可撤销工具 / 探针 / 快照及 CI 入口回退。M0-03 / M0-04 接入真实模型后必须扩充注册表、完整字段政策和兼容测试，不能删除漂移门禁来通过 CI。

来源：[Pydantic JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/)、[strict mode](https://docs.pydantic.dev/latest/concepts/strict_mode/)、[FastAPI 错误处理](https://fastapi.tiangolo.com/tutorial/handling-errors/)。API 以仓库锁定版本实测为准。
