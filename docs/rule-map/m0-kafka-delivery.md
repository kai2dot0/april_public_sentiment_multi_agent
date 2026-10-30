# M0-02B 规则—实现—验证映射

| 条款 | 实现 | 验证 / 边界 |
| --- | --- | --- |
| 大纲 §4.7.1：固定幂等配置与启动校验 | `adapters/kafka.py` | `tests/unit/test_kafka_policy.py` 的配置、缺字段与别名反例；真实客户端构造与投递 |
| arrival 逐项 ACK / 连续成功前缀 | `runtime/core/ack_prefix.py` | 三个成功回调的 6 种排列；前项/中间项失败保留后项 ACK；矛盾与未知回调拒绝 |
| Kafka 确认后才标发布 | ACK 前沿和真实 delivery callback | `test_delivery_callbacks_require_poll_and_confirm_real_records`；原型尚不写外发箱 |
| 不用 flush 队列为空冒充成功 | 真实持续丢 ACK 故障 | `test_empty_flush_queue_does_not_mean_delivered`，队列 0 但回调失败/前沿不动 |
| 分册一 §0.5.2：重试与幂等行为 | 合成消息、真实 broker、丢失已写入的 ProduceResponse | `test_lost_ack_retries_inside_producer`，幂等/关闭对照分别 1/2 条 |
| 故障接管允许重复，消费按身份去重 | 新生产者重复同一身份 | `test_application_replay_with_new_producer_can_duplicate`；未实现消费端去重/实际接管 |
| PG 事件表权威、Kafka 只传输 | 本阶段无 Kafka 事务及业务 exactly-once 声明 | 外发箱与物化事务仍待 M0-05/06 |
| 用户：PR、记录与接手 | 本功能分支、ADR、dev_memory | PR 地址和验证结果见对应 memory |
