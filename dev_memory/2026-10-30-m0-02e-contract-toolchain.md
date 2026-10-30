# M0-02E：契约工具链、Worker HTTP/JSON 与维度规模验证

日期：2026-10-30（Asia/Shanghai）。工具：Codex(GPT)。分支：`feat/m0-contract-toolchain-validation`。起点 main=`3d010c9c77710c81b0b55f2d9dec3bf198f8c4c5`，PR #5 已合并。用户要求继续下一项，本轮按 roadmap M0-02E，不提前声明完整业务契约已冻结。

## 1. 范围、依据与开发约定

依据分册一 §0.5 第 5–7 项：Pydantic → JSON Schema / 字段元数据、开放枚举线解码、Worker HTTP/JSON 或 gRPC、按大纲 §4.11 维度估计穷举规模。参考分册一 §1 / §3 / §6、总册 §3 / §8.4 / H11，以及大纲 §4.1、§4.7、§4.9、§4.11。

交付技术原型与可复现证据；正式 FetchPage / 执行上下文 / 模式权限 / 错误等仍属 M0-03，完整结果包投影 / 签名仍属 M0-04，持久化控制接线仍待 M0-05 / M0-06。本轮未实现 18 行动作表、完整 V1–V19 判定或 H11 验收，不把部分探针当作业务完成。

遵循独立功能分支 → 验证 → memory → PR → 最终提交 CI → merge，不直接向 main 提交。无子 Agent，没有 .codegraph/，未建索引；shell 均用 RTK 前缀。应用 ponytail 原则，复用已有 app 工厂、Pydantic / FastAPI / httpx / uvicorn / pytest / JCS，不增加依赖。

新增共享 contracts 工具按总册 §11.1 三级影响信息在 ADR 0006 / PR 留痕；尚无原有跨组业务调用方或兼容迁移。HTTP/JSON 属推荐内一级选型，无 A/B/C 裁剪。不更改 HLD 原件，不虚构团队接口签署已发生；两批正式冻结状态保持待完成。

## 2. 核心文件和模型唯一来源

| 文件 | 职责 |
| --- | --- |
| `contracts/tooling.py` | WireModel、field_metadata、checked_schema、decode_wire；共享叶子工具，不依赖 runtime / app |
| `tests/support/contract_probe.py` | 测试专用模型、Literal 已知域、V1 位置探针、独立 HTTP app |
| `scripts/check_contract_schemas.py` | 模型注册表 → 稳定 schema 快照；--write 显式写，普通检查只读 |
| `contracts/schemas/probes/*.schema.json` | 三个模型的导出快照，目录 README 明示不是正式业务契约 |
| `tests/support/decision_matrix.py` | 显式维度 product、唯一排除、固定原始序号分片和计数 |
| `scripts/measure_contract_matrix.py` | 可保存 / 打印机器环境、时间戳、维度、排除理由和耗时证据 |
| `docs/validation/m0-02e-matrix-scan.json` | 本机最终完整扫描实测，包含绑定维度顺序的 profile_digest |
| 两份新 unit 测试 | 模型 / schema / 真实 TCP 43 项 + 规模 / 分片 9 项，共 52 项 |

探针模型只存在 tests/support，PROBE_MODELS 为唯一导出注册表：PageWireProbe、StrictStatusProbe、WorkerEnvelopeProbe。OutcomeProbe / TombstoneProbe 作为嵌套定义自动导出，不手写 schema。没有将它们加入生产 app、SDK 或正式 Worker 协议。

## 3. 字段政策、严格字节入口与数值边界

WireModel：strict=True、extra=forbid、validate_default=True、allow_inf_nan=False、hide_input_in_errors=True。field_metadata 返回 digest include/exclude、default_semantics、routable；checked_schema 验证所有嵌套声明属性，拒绝子模型额外字段或输入 / 输出 schema 异形（例如 Field(exclude=True)）。本探针只处理同形 JSON 线模型，不声称覆盖所有 Pydantic 类型和 validator 兼容性。

字段元数据只声明政策，**没有实现按字段投影**；lease_token 标记 exclude 仅验证元数据导出，不代表当前全值 hash 会忽略它。完整结果包的排除白名单与语义默认值必须在 M0-04 明确，不能据此允许插件排除任意字段。

decode_wire 保持 JSON int 精度：先严格 UTF-8 解码，stdlib json.loads 配 object_pairs_hook 拒绝重复键，parse_constant 拒绝 NaN / Infinity，再检查 Unicode 可编码和有限数值，最后 model_validate(strict=True)。不使用 JCS 文本入口，避免 binary64 舍入；错误为固定 WireDecodeError 并抑制异常链，不暴露来源值或 Pydantic input / ctx。

默认值明示：页内 optional 原因缺省为 null，条目结果 / 墓碑数组缺省为空数组；model_fields_set 保留缺省来源，dump 时显式展开。全值往返 hash 测试比较展开后的模型，不把稀疏原始请求与展开请求混为同一字节。

实验发现并保留边界：lease_generation 探针可精确保留 `2**63-1`，但这个值进入既有 JCS 对象入口会被拒绝；它不是安全整数。subjectseq 以字符串保留 `18446744073709551615`。M0-03 必须决定正式代次 / ID 的安全数值范围或字符串形式，M0-04 再加 JCS 准入；不得先转 float“修复”。当前没有真实 lease 服务依赖该探针数值政策。

## 4. 未知枚举与 HTTP/JSON 结论

已知域以 Python 3.12 Literal 别名集中声明，ENUM_DOMAINS 从别名的 __value__ 提取，不重复维护一份枚举表。线模型对 status / completeness / end_reason / failure_scope / status_reason / item_outcomes.outcome 都用原生字符串，未知 / 空 / 带空白原值不 trim、不转 unknown、不在解码阶段拒绝。构造侧 StrictStatusProbe 对照使用 Literal，拒绝未知值。

unknown_enum_paths 仅标记未知位置；status_reason 在 forbidden 下检查已知原因，其他状态下是自由诊断。它不是完整 verdict，不计算 primary，不验证控制权限，不登记墓碑。真实往返证明未知 status 与 outcome 不会导致旁边的合成 tombstones 载荷消失；控制的受限写入仍须后续真实协议与授权。

选择 HTTP/JSON（HLD 推荐），复用现有 FastAPI / Pydantic / httpx；没有引入 gRPC / protobuf / registry，也没有对 gRPC 做吞吐性能比较。测试 app 从生产 create_app 工厂扩展，额外挂载 `/_probe/worker` 的 codec echo。生产 create_app 的 OpenAPI 中无此路径，测试 200 只表示编解码完成。

真实 TCP fixture：先 bind loopback:0 取得独占 socket，传给 uvicorn；startup 完成后 Event 通知客户端，不用随机 sleep 抢端口；httpx trust_env=False，禁环境代理；关闭客户端后 should_exit，线程有界 join，必要时 force_exit，断言线程与异常列表均干净。没有容器 / 数据库依赖。

测试入口中间层先检查 JSON Content-Type 和严格字节结构，再交给正常 Pydantic 路由及 response_model，保留自动 OpenAPI。64 KiB 测试阈值在读取 body 后检查，不能当成已完成的流式内存限额；正式 HTTP 认证、lease / fencing、资源准入和业务幂等仍待正式协议服务。

## 5. 维度轮廓、全扫与 CI 设计

按大纲 §4.11 的 bounded_slice 维度，大小依次为：调度 2、status 12、status_reason 5、failure_scope 4、completeness 4、end_reason 6、continuation 2、有效前缀 2、单资源 2、item_outcomes 聚合类 9、请求方式 2、控制校验结果 2。

item_outcomes 先取 HLD 的五类：空、集合内仅 present / failed、集合内含 missing、集合外不含 missing、集合外含 missing；再扩展未知 outcome 的集合内 / 外 × 含 / 不含 missing 四类，以覆盖 V1 的未知 outcome 与关系组合。自由字符串只用语义代表值，不对任意内容作无限穷举。

调度 / 请求方式来自已校验平台上下文，未知代表值只针对 V1 页面枚举。正式判定器接入前复核聚合类是否足够，特别是 V8 查询页 failed / missing 与观测集合的区分；若需细分或上下文域改变，要重新生成轮廓与计数，本轮数字不是最终 H11 固定规模。

显式 itertools.product 生成 **3,317,760** 个原始组合。仅排除 single_resource AND request_mode=query_or_stream 共 **829,440**，不给 verdict；其余 **2,488,320** 全部扫描。没有将其他语义违规组合筛掉。未知枚举探针命中 **1,915,920**；该数不是完整违规组合数。

分片按原始 ordinal % shard_count，不能先删不可达组合再重编号。三片的 selected / excluded / enumerated / unknown / checksum 之和等于全量；checksum=`4127807278080`。profile_digest 对有序的 [维度名, 取值数组] 列表做 JCS SHA-256，绑定顺序，避免对象键排序掩盖维度重排；最终值 `ca4507e30c68e3fbaece6d1ec9a29fe6fa95cc1347451e62bd7f8aba6ec5c136`。

最终记录于 2026-10-30 22:48:52（Asia/Shanghai）：Darwin ARM64、Python 3.12.12、Pydantic 2.14.2，完整扫描约 **0.409 秒**。没有保存数百万组合对象，只保留流式计数。计时仅含扫描与未知值检查，不含完整 V1–V19、动作匹配或事务，是后续性能的下界，**不是 H11 通过证据**。

当前 make check 每 PR 全扫；CI checks 新增 make measure-contract-matrix，在日志输出 Linux 计数 / 耗时。当前无需只为扫描新增夜间任务。正式判定器接入后，要用同一真实纯函数重测：规则 / 清单 / 判定器改变跑全量，其他 PR 可确定性分片、夜间全量，再按真实耗时决定并行度；不能把本次 0.409 秒推广为完整判定器成本。

## 6. 验证、命令与交付证据

实际已完成：

- `make test-contract-toolchain`：52 passed；真实 HTTP 使用 loopback 网络，未用 ASGITransport 冒充网络证明。
- 最终 `make check`：340 passed、29 个集成用例按 marker 排除；总收集 369 项。8 条 import-linter 契约、迁移链 1 revision、锁文件和 3 份 schema 门禁通过。
- 模型 / 元数据 / schema 的缺失、漂移、未登记、嵌套漏标、子模型额外字段和静默排除均有反例；检查只读，不自动修复。
- TCP 覆盖未知枚举 + 墓碑保留、默认展开、Content-Type / 请求阈值、精确数值字符串、结构错误、重复 / 转义重复键、NaN、代理项、UTF-16 / 非法 UTF-8 / BOM、固定不含 input 的错误、OpenAPI 和生产路由隔离。
- 已独立执行完整维度测量并提交 JSON 实测；无新依赖、数据库迁移或容器拓扑改变。
- 7 个新 Python 文件的 AI header / 日期、函数 docstring、AST 语法检查与 `git diff --check` 通过。

复现：

```bash
rtk proxy make setup
rtk proxy make schema-check
rtk proxy make test-contract-toolchain
rtk proxy make check
rtk proxy make measure-contract-matrix
# 需要更新快照 / 基准时显式执行并审查 diff：
rtk proxy make schema-write
rtk proxy env UV_CACHE_DIR=.cache/uv UV_PYTHON_INSTALL_DIR=.cache/python uv run --locked python -m scripts.measure_contract_matrix --output docs/validation/m0-02e-matrix-scan.json
# 分片例子：
rtk proxy env UV_CACHE_DIR=.cache/uv UV_PYTHON_INSTALL_DIR=.cache/python uv run --locked python -m scripts.measure_contract_matrix --shard-index 0 --shard-count 3
```

远端仍须跑完整 PG / Kafka / observability 回归；本轮未重复扰动本地容器。交付 [PR #6](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/6)，实现提交 `9affd27`；最终提交的 CI（含 Linux 扫描测量）、merge SHA 与合并状态以该 PR 记录为准，全部通过后才合并，不把未执行项记作通过。

## 7. 接手顺序、范围边界与回退

先读本记录、ADR 0006 与 `docs/rule-map/m0-contract-toolchain.md`；再看共享 tooling、探针模型与 schema 注册表、真实 HTTP 测试、维度定义 / 基准及 CI。新增代码 7 个文件均按用户要求保留 AI header 和函数说明；生成 JSON 不加注释。

下一项 M0-02F：进程组、cgroup、网络与缓存清理验证；随后 M0-02G 来源协议核实。进入 M0-03 / M0-04 时：把完整模型纳入正式注册表，冻结字段和默认语义、数值域、错误映射、版本演进及摘要 include/exclude 政策；再实现实际投影、签名和提交。不要把 Probe 模型重命名就当完整 FetchPage；不要在生产 API 挂载 echo 路由。

完整 H11 必须补真实 18 行 / 19 条 YAML 与同一判定纯函数，逐组合输出 verdict / 全部违规码 / primary / 匹配行数 / 最终动作，并对排除项提供构造断言；本次只有维度成本基线。schema diff 也不能替代 validator 语义兼容测试、控制鉴权和事务验收。

当前可撤销工具 / 探针 / 快照、Make / CI 和文档入口回退，无业务数据清理，无 SDK 兼容迁移。后续有实际跨组调用和持久数据后必须按兼容变更处理；不能禁用 schema 检查或改为静默丢字段来规避失败。

## 8. 合并证据（下一项开始时补录）

PR #6 已于 2026-10-30 22:58（Asia/Shanghai）合并，merge SHA=`2b05440d5b0248b36472e3b849a90442873865ae`。最终提交 `5ba7d40b14e71f09bf813eb257b65ed8664abfce` 的 CI run `41076041357` 中 checks / infrastructure 均 SUCCESS；远端 340 项通过，Linux 扫描 0.767 秒、计数和有序 profile digest 与本地一致。合并后本地 main 干净，文件树与已检查提交一致。
