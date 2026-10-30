# ADR 0003：M0-02B Kafka 幂等生产者与 ACK 验证

日期：2026-10-30。依据：大纲 §4.7.1、分册一 §0.5.2 / §5、分册二 §0.4 / §1。新增依赖使用 HLD 推荐的 confluent-kafka，本 PR 按总册 §11.1 的新增依赖变更信息登记，不更改跨组契约或冻结语义。

## 客户端与启动策略

选择并锁定 confluent-kafka 2.16.1（本地 librdkafka 2.16.0）。最低版本与实际使用的 Producer.close API 对齐；不引入 Kafka 事务或其他编排框架。

`adapters/kafka.py:create_producer` 在构造原生客户端前显式校验：bootstrap 配置、`enable.idempotence=true`、`acks=all`、正整数 retries、1–5 个 in-flight、成功 delivery report 不关闭。相关别名拒绝与规范字段并存，避免隐藏覆盖。acks 当前采用唯一规范写法 `all`，不静默转换配置类型或依赖客户端自动纠正。

## ACK 前沿原型

`runtime/core/ack_prefix.py:AckPrefix` 绑定一个有序唯一身份批次，保存逐项终态，只输出连续成功前缀；前项失败不将已成功的后项改为失败。重复一致回调幂等，矛盾回调或未知身份明确报错。

恢复 suffix 从首个未确认位置开始，可能包含后面已 ACK 的条目；这类重发由未来消费端按事件身份去重。原型仅由拥有者线程更新，不是持久化权威状态，也不实现桶租约/代次 CAS、停止/接管或外发箱事务。

## 可重复的 ACK 丢失实验

直接 bootstrap 到代理还不够：客户端会按 broker advertised listener 建立后续连接。因此提供测试专用 Compose override，把同一开发 broker 的 HOST advertised 地址指向 loopback 19093；代理从该端口转发到原 broker 19092。容器内 INTERNAL 地址保持不变。

代理只在测试进程中运行，按每条连接的 ApiKey=0 / correlation id 识别真实 ProduceResponse，不模拟 broker。第一次响应可用命名暂停点截住：独立消费者先确认消息已写入，再放行丢弃响应并断开连接，触发同一次 produce 的客户端内部重试。所有帧只在内存转发，不打印正文，帧大小与网络等待有界。

| 场景 | 实测断言 |
| --- | --- |
| 正常投递 12 条 | 入队不触发 Python delivery callback，poll/flush 后逐项成功；同分区记录/offset 顺序与发送一致 |
| 丢失一次 ACK，启用幂等 | 实际 Produce 请求至少两次，但完整分区仅 1 条记录 |
| 同样丢 ACK，关闭幂等的对照组 | 完整分区出现 2 条；对照直接用原生客户端，平台入口拒绝此配置 |
| 持续丢 ACK | flush 返回剩余 0，但回调超时失败、确认前沿不动，broker 仍已有 1 条记录 |
| 新生产者重发同一身份 | 两条记录、offset 0/1，证明应用重发不由客户端内部幂等跨实例消除 |

消费按 broker high watermark 读取完整临时分区，不以短暂空 poll 当作“没有重复”。不使用真实来源数据或业务 topic，不提交消费组位点；每个用例只创建/删除自己的随机 topic。测试先核对本机单 broker 的 advertised 地址，未配置测试拓扑时在建 topic 前失败。

## 执行与恢复

```bash
rtk proxy make kafka-test-up
rtk proxy env TEST_KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:19092 TEST_KAFKA_PROXY_PORT=19093 make test-kafka
rtk proxy make infra-up
```

仅对本项目可丢弃开发 broker 运行。`kafka-test-up` 会重启本项目 Kafka 并临时改变 HOST advertised 地址；实验结束即用 `infra-up` 恢复普通拓扑，保留卷。代理端口不可与 upstream 相同，不并行运行共用同一端口的故障套件。

原 `test-integration` 保持数据库集成入口（排除 kafka marker）；Kafka 用独立 `test-kafka`。CI 依次运行两者，防止只验证 PG 就宣称 Kafka 通过。普通和故障 Compose 都进入配置检查。

## 未覆盖范围

只有单 broker、单分区 PLAINTEXT 合成实验；acks=all 在此只有一个 ISR，不代表多副本容灾验证。未实现 PG 外发箱、桶 fencing、消费者物化事务、毒行隔离或 exactly-once 业务结果。回调顺序观察只适用于该分区实验，乱序安全性由独立的六种回调排列与失败反例验证。

资料：[confluent-kafka API](https://docs.confluent.io/platform/current/clients/confluent-kafka-python/html/index.html)、[librdkafka 配置](https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md)、[Kafka 协议](https://kafka.apache.org/41/design/protocol/)。用法以锁定版本实测为准。
