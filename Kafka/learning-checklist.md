# Kafka 学习检查清单

这份清单用于跟踪整套教程的学习进度。每完成一项，建议在旁边写下实验日期、遇到的问题和你的总结链接。

## 第 0 阶段：准备阶段

- [ ] 能用 Docker Compose 启动 Kafka。
- [ ] 能进入 Kafka 容器执行命令。
- [ ] 能创建 topic。
- [ ] 能用命令行 producer 发送消息。
- [ ] 能用命令行 consumer 消费消息。
- [ ] 能查看 consumer group 的 offset 和 lag。
- [ ] 能解释为什么 consumer 不应该在业务成功前提交 offset。

## 第 1 阶段：Kafka 核心概念

- [ ] 能解释 broker、topic、partition、replica。
- [ ] 能解释 leader、follower、ISR。
- [ ] 能解释 offset 为什么只在 partition 内有意义。
- [ ] 能解释 offset 为什么属于 consumer group。
- [ ] 能说明 consumer 数量超过 partition 数量会发生什么。
- [ ] 能解释 retention 和 log compaction 的区别。

## 第 2 阶段：Producer 与 Consumer

- [ ] 能为订单、用户、库存事件选择合适的 message key。
- [ ] 能解释 `acks=0`、`acks=1`、`acks=all`。
- [ ] 能说明 batch、linger、compression 如何影响吞吐和延迟。
- [ ] 能说明自动提交 offset 的风险。
- [ ] 能设计手动提交 offset 的消费流程。
- [ ] 能解释 rebalance 的触发原因。
- [ ] 能根据 lag 现象提出排查路径。

## 第 3 阶段：Go 客户端开发

- [ ] 已选择一个主线 Go Kafka 客户端。
- [ ] 已设计 producer wrapper 接口。
- [ ] 已设计 consumer handler 接口。
- [ ] 能在 Go 服务中使用 context 控制超时和退出。
- [ ] 能记录 topic、partition、offset、key、event_id。
- [ ] 能设计 retry 和 DLQ 的代码边界。
- [ ] 能画出项目目录结构。

## 第 4 阶段：可靠性与一致性

- [ ] 能解释 at-most-once、at-least-once、exactly-once。
- [ ] 能说明 Kafka exactly once 和业务 exactly once 的区别。
- [ ] 已设计 `processed_events` 去重表。
- [ ] 能使用业务唯一约束或状态机做幂等。
- [ ] 已设计 retry topic。
- [ ] 已设计 DLQ 消息结构。
- [ ] 能解释 outbox pattern 解决什么问题。

## 第 5 阶段：性能与可观测性

- [ ] 已完成 producer 压测。
- [ ] 已完成 consumer 压测。
- [ ] 已对比不同压缩算法。
- [ ] 已观察 consumer lag 增长和下降。
- [ ] 已设计 Prometheus 指标。
- [ ] 已设计 lag、DLQ、错误率告警。
- [ ] 能写一份完整压测结论。

## 第 6 阶段：生产运维与架构设计

- [ ] 能为业务 topic 设计 partition 数量。
- [ ] 能估算 topic 存储容量。
- [ ] 能解释 replication factor 和 min ISR 的意义。
- [ ] 能列出服务需要的 Kafka ACL。
- [ ] 能设计事件驱动架构图。
- [ ] 能说明最终一致性和补偿流程。
- [ ] 能设计事件 schema 版本演进策略。

## 第 7 阶段：项目实战

- [ ] 已完成项目 README。
- [ ] 已完成 Docker Compose。
- [ ] 已完成 topic 创建脚本。
- [ ] 已完成订单创建链路。
- [ ] 已完成库存消费和幂等扣减。
- [ ] 已完成 retry topic。
- [ ] 已完成 DLQ。
- [ ] 已完成 outbox worker。
- [ ] 已完成消费指标和结构化日志。
- [ ] 已完成正常链路测试。
- [ ] 已完成重复消费测试。
- [ ] 已完成消费失败重试测试。
- [ ] 已完成 DLQ 测试。
- [ ] 已完成优雅退出测试。

## 最终自测问题

如果下面的问题都能回答清楚，说明你已经具备 Go 后端 Kafka 项目开发的基本能力：

1. 为什么 Kafka 的 partition 决定 consumer group 的最大并行度？
2. 为什么 Kafka consumer 必须考虑重复消费？
3. 什么时候可以提交 offset？
4. retry topic 和 DLQ 分别解决什么问题？
5. outbox pattern 为什么能改善数据库和 Kafka 的一致性？
6. consumer lag 持续升高应该怎么排查？
7. 事件 schema 如何做到向后兼容？
8. Go 服务收到 SIGTERM 时，Kafka consumer 应该如何退出？

