# M0-02B：Kafka 幂等生产者、ACK 与重试验证

日期：2026-10-30（Asia/Shanghai）。工具：Codex(GPT)。分支：`feat/m0-kafka-delivery-validation`。起点：M0-02A 已通过 PR #2 合并，main=`7b6f76d2f6bcf35219f7a2a35efc15707c15c0cf`。

## 1. 背景与范围

用户要求继续下一项，按 roadmap 进入 Kafka 验证。依据：大纲 §4.7.1；分册一 §0.5.2、§5；分册二 §0.4、§1。完整交付客户端启动策略、ACK 连续前沿原型和真实重试实验，保留独立 PR / memory 流程。无子 Agent，无 CodeGraph 索引；采用 ponytail 原则复用现有 pytest、Compose、命名 PausePoint 和 CI，不引入额外故障测试框架。

本阶段不实现外发箱表、桶租约/代次 CAS、消费者 PG 物化、死信或业务 exactly-once。M0-02 的其他分项仍按 roadmap 独立交付。

## 2. 依赖与启动入口

- 新依赖 confluent-kafka 是 HLD 推荐客户端，选定 2.16.1，实际 librdkafka 2.16.0。Python 环境仍为 3.12.12。pyproject/uv.lock 同步锁定，新增依赖按总册 §11.1 变更信息在本 PR 登记，无跨组接口变更。
- 版本下限设 2.16.1，与已实测 Producer.close API 对齐；close 在回调之外调用，避免客户端警告的回调内关闭死锁。
- `adapters/kafka.py:validate_producer_config` 要求显式 bootstrap、true 幂等、acks=all、正整数 retries、1–5 in-flight、成功 delivery report 开启。bool 不能冒充整数。拒绝可覆盖这些设置的别名，避免客户端自动调整掩盖配置缺口。
- `create_producer` 校验后返回原生 Producer，不创建新的生产者包装类，不把入队/flush 结果转换成业务成功。
- 禁止 Agent / SDK / 连接器直接导入驱动的已有 import-linter 规则继续有效。关闭幂等的对照组只存在于测试 helper，正式入口不能创建这种客户端。

## 3. ACK 前沿原型

`runtime/core/ack_prefix.py:AckPrefix` 初始化有序、非空字符串且批内唯一的事件身份；空批次允许，无序集合/字符串/重复身份拒绝。保存逐项最终结果，只有连续成功前缀可确认。

- 成功后项不会因前项失败被覆盖。`acknowledged_ids` 保留全部已成功身份，`confirmed_ids` 只给连续前缀，`replay_suffix` 从第一项未确认位置开始。
- 同一最终回调重复且一致时幂等；未知身份、相互矛盾回调、非 bool 结果报错，异常不回显消息内容。
- suffix 可包含已 ACK 的后项，重发仍需未来消费端按事件身份去重。
- 一个对象对应单批次/代次，仅拥有者线程更新；只是内部算法原型，不是进程内权威状态。后续正式外发箱必须以 PG 逐项 ACK 或前缀、桶代次和租约做 CAS，不能直接以这个对象确认持久化。

## 4. 真实 ACK 丢失与拓扑

### 测试 listener

仅把 bootstrap 指向代理不够，客户端接下来会按 advertised 地址绕过它。`deploy/kafka-fault-test.yaml` 作为 Compose override，将本项目 broker 的 HOST advertised 指向 `127.0.0.1:19093`，真正 broker 入口仍为 `19092`；INTERNAL 地址不变。

`make kafka-test-up` 仅对本项目可丢弃开发 Kafka 重启并启用该拓扑，`make infra-up` 恢复普通开发拓扑，不删卷。共享同一端口的故障套件不能并行执行，也不能在运行实际采集的 broker 上执行此操作。

### 帧代理

`tests/support/kafka_proxy.py:ProduceAckProxy` 为本机 PLAINTEXT 测试专用，按 Kafka int32 长度分帧；帧上限 16 MiB。请求 ApiKey=0 表示 Produce，按每条连接的 correlation id 将实际响应归类。

故障命中真实 ProduceResponse 后可以先通过已复用的命名 PausePoint 暂停。测试使用独立消费者读到 offset=0，证明 broker 已存储，且 producer.poll(0) 仍无最终回调；再放行代理丢 ACK、关闭连接，触发原生客户端内部重试。不是伪造 broker、重新调用 produce 或按空 poll 推断重试成功。

代理不打印帧/正文，不读取业务数据，所有等待有界。清理释放命名暂停、关闭 listener 和连接、join 线程；异常保留为测试失败。采用线程只是测试工具，不向应用层开放 TCP 绕过通道。

### 临时资源

`tests/integration/test_kafka_delivery.py:kafka_probe` 要求显式 TEST_KAFKA_BOOTSTRAP_SERVERS，限定 loopback upstream 与不同代理端口。先核对 broker advertised listener 确实为测试代理，未启用拓扑时在建 topic 前失败。

每个测试创建自己的随机 `m0-kafka-probe-<uuid>` 单分区 topic，清理仅删本次 topic。`KafkaProbe.records` 根据 broker high watermark 读取完整分区；不将短暂空 poll 当作无重复。Consumer 使用手工分区分配，不自动提交/存储消费位点。Producer 先有界 flush，必要时 purge，再在回调外 close。

## 5. 验证场景与结果

| 场景 | 实际断言 |
| --- | --- |
| 12 条正常投递 | produce 只入队、无 Python 回调；flush/poll 后 12 次成功，offset 0–11，完整读取顺序一致 |
| 同一次 produce 丢一次已写入 ACK，幂等开启 | 实际 Produce 请求至少 2 次；只 1 次最终回调、完整分区只有 1 条 |
| 同上但关闭幂等对照 | 完整分区 2 条，证明故障确实能触发重复写入 |
| 持续丢 ACK 至 message timeout | flush 返回剩余 0、回调 _MSG_TIMED_OUT、前沿不动；broker 已有 1 条，说明失败并不证明未写入 |
| 新生产者重发同一事件 | 两条记录 offset=0/1；客户端内部幂等不能跨应用重发按事件 ID 去重 |

本地已完成：

- `make check`：最终 89 项单元/架构用例通过；8 条导入契约、迁移链和锁文件检查通过。
- `make test-kafka`：5 项真实 Kafka 用例通过（包含幂等/关闭对照两个参数用例）。
- `make test-integration`：原 24 项真实 PG 用例仍通过；总收集 118 项（89 单元 + 24 PG + 5 Kafka）。
- `make compose-check`：普通与故障 override 配置均通过。
- 实验后元数据只读检查：临时测试 topic 剩余 0；默认开发 advertised 端口已恢复为 19092，core 服务 healthy。
- 新增 6 个 Python 文件及故障 YAML 的 header / Python 语法、git diff --check 均通过。

交付 PR：[PR #3](https://github.com/kai2dot0/april_public_sentiment_multi_agent/pull/3)。实现提交 `a70e4e5`，最终 CI、合并状态和 merge SHA 以该 PR 的 GitHub 记录为准；全部检查通过后才合并，无直接向 main 提交。

## 6. CI、命令与接手

- `make test-integration` 继续作为数据库集成入口，表达式为 integration and not kafka；避免只提供 PG 配置就隐式触发 broker 故障。
- `make test-kafka` 独立需要 TEST_KAFKA_BOOTSTRAP_SERVERS；TEST_KAFKA_PROXY_PORT 默认 19093，若修改 .env 中该值，执行测试也需导出同值。
- CI infrastructure job 在 PG/smoke-infra 之后执行 kafka-test-up 和 test-kafka，再恢复普通 Compose 配置启动 observability 并验证。两类集成都必须通过。
- 本轮曾因 unit 与 integration 同名 test_kafka_delivery.py 导致 pytest 收集冲突，改为 test_kafka_policy.py，未引入额外 import 模式或插件。

复现：

```bash
rtk proxy make setup
rtk proxy make check
rtk proxy make kafka-test-up
rtk proxy env TEST_KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:19092 TEST_KAFKA_PROXY_PORT=19093 make test-kafka
rtk proxy make infra-up
```

关键入口：`adapters/kafka.py`、`runtime/core/ack_prefix.py`、`tests/support/kafka_proxy.py`、`tests/support/kafka.py`、两份 Kafka 测试。设计与规则映射分别为 `docs/decisions/0003-kafka-delivery.md`、`docs/rule-map/m0-kafka-delivery.md`。

## 7. 边界与后续

单 broker 的 acks=all 只有单 ISR，不验证多副本/跨机容灾。单分区正常回调观察不能推广为跨分区全局顺序；乱序安全由六种独立 ACK 排列及前/中间失败反例覆盖。无真实外发箱、消费者 PG 去重、桶接管、网络外部副作用或模型数据。

下一项按 roadmap 为 M0-02C RFC 8785/JCS 测试向量与选型。正式提交/发布需结合已有 PG 并发实验、契约、签名和回执实现，不把本原型当作完整 M0 硬判据证据。后续同模块迭代继续更新本文件。
