# Go 后端工程师 Kafka 系统学习路线图

> 目标：从“会启动 Kafka、会收发消息”进阶到“能在 Go 后端项目中设计、开发、排查和优化 Kafka 消息系统”。

## 适合人群

- 已掌握 Go 基础语法、goroutine、channel、context、HTTP/RPC、数据库基本使用。
- 想把 Kafka 用在订单、支付、日志、事件驱动、异步任务、实时数据同步等后端场景。
- 希望学习路线既覆盖底层原理，也能落到 Go 项目代码与生产实践。

## 学习总览

建议周期：8 到 12 周。

| 阶段 | 主题 | 目标 |
| --- | --- | --- |
| 第 0 阶段 | 前置准备 | 补齐分布式系统、Go 并发、Docker、Linux 基础 |
| 第 1 阶段 | Kafka 入门与核心概念 | 理解 broker、topic、partition、offset、consumer group |
| 第 2 阶段 | Producer 与 Consumer | 掌握可靠投递、批量发送、消费提交、重平衡 |
| 第 3 阶段 | Go 客户端开发 | 使用 Go 编写生产者、消费者、消费者组、管理脚本 |
| 第 4 阶段 | 可靠性与一致性 | 掌握幂等、事务、顺序性、重复消费、消息丢失处理 |
| 第 5 阶段 | 性能与可观测性 | 学会压测、调参、监控 lag、定位吞吐与延迟问题 |
| 第 6 阶段 | 生产运维与架构设计 | 掌握容量规划、安全、故障恢复、事件驱动架构 |
| 第 7 阶段 | 项目实战 | 独立完成一个接近真实后端业务的 Kafka 项目 |

## 第 0 阶段：前置准备

时间建议：3 到 5 天。

### 必备基础

- Go：
  - goroutine、channel、select
  - context 取消与超时控制
  - error wrapping 与日志规范
  - graceful shutdown
  - 单元测试、集成测试、benchmark
- Docker：
  - Docker Compose
  - 容器网络
  - volume 持久化
- Linux 与网络：
  - TCP 基础
  - 文件描述符
  - 磁盘 IO
  - 常用排查命令：`netstat`、`ss`、`lsof`、`top`、`iotop`
- 分布式系统：
  - CAP 的基本含义
  - 副本、leader、quorum
  - at-most-once、at-least-once、exactly-once 的区别

### 产出

- 能用 Docker Compose 启动一个 Kafka 环境。
- 能解释“为什么消息队列可以削峰、异步、解耦”。

## 第 1 阶段：Kafka 核心概念

时间建议：1 周。

### 学习内容

- Kafka 是什么：
  - 分布式事件流平台
  - 日志型存储
  - 发布订阅与消息队列模型的结合
- 核心组件：
  - broker
  - topic
  - partition
  - replica
  - leader 与 follower
  - offset
  - consumer group
  - controller
- 存储模型：
  - 顺序写日志
  - segment 文件
  - retention
  - log compaction
- 集群模式：
  - KRaft
  - ZooKeeper 历史模式，只了解即可

### 必做实验

1. 使用 Docker Compose 启动 Kafka。
2. 创建 topic：设置 3 个 partition、1 个副本。
3. 使用命令行 producer 写入消息。
4. 使用命令行 consumer 读取消息。
5. 创建多个 consumer，观察 consumer group 中 partition 的分配变化。
6. 修改 retention 配置，观察消息过期。

### 必须能回答的问题

- topic 和 partition 的关系是什么？
- partition 为什么是 Kafka 并行度的核心？
- 一个 consumer group 里 consumer 数量大于 partition 数量会发生什么？
- offset 存在哪里？提交 offset 的意义是什么？
- Kafka 为什么能做到高吞吐？

## 第 2 阶段：Producer 与 Consumer 机制

时间建议：1 到 2 周。

### Producer

重点理解：

- key 如何决定 partition
- batch.size、linger.ms、compression.type
- acks=0、acks=1、acks=all 的区别
- retries 与 delivery.timeout.ms
- enable.idempotence
- max.in.flight.requests.per.connection
- 消息顺序性的边界
- callback / delivery report

必须掌握：

- 如何避免消息丢失
- 如何提高吞吐
- 如何处理发送失败
- 如何设计 message key
- 如何设计消息 header

### Consumer

重点理解：

- poll/fetch 模型
- auto commit 与 manual commit
- consumer group rebalance
- session.timeout.ms
- heartbeat.interval.ms
- max.poll.interval.ms
- lag 的含义
- pause/resume

必须掌握：

- 如何保证业务处理成功后再提交 offset
- 如何处理重复消费
- 如何处理单条消息失败
- 如何优雅关闭 consumer
- 如何避免 consumer 长时间处理导致 rebalance

### 必做实验

1. 创建 1 个 producer，3 个 consumer，观察分区分配。
2. 手动关闭某个 consumer，观察 rebalance。
3. 模拟消费失败，比较自动提交与手动提交的差异。
4. 设置不同 key，观察消息进入不同 partition。
5. 批量发送 10 万条消息，对比是否开启压缩的吞吐差异。

## 第 3 阶段：Go 客户端开发

时间建议：2 周。

### Go 客户端选择

常见选择：

| 客户端 | 特点 | 适合场景 |
| --- | --- | --- |
| `confluent-kafka-go` | 基于 librdkafka，能力完整，性能强 | 公司使用 Confluent、重视性能、需要成熟生态 |
| `franz-go` | 纯 Go，功能完整，API 现代 | 希望减少 C 依赖、偏 Go 原生体验 |
| `sarama` | 老牌纯 Go 客户端，社区使用广 | 维护旧项目、已有 Sarama 基础设施 |

建议学习顺序：

1. 优先用 `franz-go` 或 `confluent-kafka-go` 做新项目。
2. 了解 `sarama`，因为很多公司老项目仍在使用。
3. 不要一开始同时深入三个客户端，先选一个做完整项目。

### Go Producer 必会代码能力

- 初始化 producer 配置
- 支持 context 超时
- 同步发送与异步发送
- 发送结果回调
- 配置重试、压缩、acks、幂等
- 对消息 key、value、header 做统一封装
- 优雅关闭 producer

### Go Consumer 必会代码能力

- 加入 consumer group
- 手动提交 offset
- 批量消费
- 单条失败处理
- 失败重试
- 死信 topic
- pause/resume
- graceful shutdown
- 暴露 Prometheus 指标

### 推荐代码结构

```text
internal/
  kafka/
    producer.go
    consumer.go
    config.go
    message.go
    retry.go
    metrics.go
  order/
    handler.go
    event.go
cmd/
  producer-demo/
  consumer-demo/
docker-compose.yml
```

### 必做小项目

项目名：订单事件异步处理系统。

功能：

- HTTP API 创建订单。
- 订单服务写入数据库后发送 `order.created` 事件。
- 消费者监听 `order.created`，模拟：
  - 扣库存
  - 发优惠券
  - 发送通知
- 消费失败时进入 retry topic。
- 多次失败后进入 dead letter topic。
- 暴露消费 lag、处理耗时、失败次数指标。

## 第 4 阶段：可靠性、一致性与消息语义

时间建议：1 到 2 周。

### 核心主题

- 消息丢失的来源：
  - producer 发送失败
  - broker 未刷盘或副本不足
  - consumer 提前提交 offset
- 重复消费的来源：
  - consumer 处理成功但提交 offset 失败
  - rebalance
  - producer retry
- 顺序性：
  - 单 partition 内有序
  - 多 partition 全局无序
  - 相同业务 key 进入同一 partition
- 幂等：
  - producer 幂等
  - consumer 业务幂等
- 事务：
  - transactional.id
  - read_committed
  - consume-transform-produce 场景

### Go 后端重点

你真正要掌握的是“业务可恢复”，而不是只记 Kafka 配置。

建议实践：

- 消费者处理消息时使用业务唯一键去重。
- 业务表增加 `event_id` 或 `message_id` 去重记录。
- 对外部副作用操作使用幂等键，例如支付回调、优惠券发放、库存扣减。
- 失败消息不要无限重试，使用 retry topic 和 dead letter topic。
- 关键链路保留完整日志：topic、partition、offset、key、event_id。

### 必须能回答的问题

- Kafka 能不能保证 exactly once？
- Kafka 的 exactly once 和业务 exactly once 是一回事吗？
- 为什么 consumer 一定要考虑幂等？
- 什么场景必须保证同一 key 的消息顺序？
- retry topic 与死信 topic 应该怎么设计？

## 第 5 阶段：性能调优与可观测性

时间建议：1 周。

### Producer 调优

重点参数：

- batch.size
- linger.ms
- compression.type
- acks
- retries
- buffer.memory
- max.in.flight.requests.per.connection

调优方向：

- 追求吞吐：增大 batch、适当 linger、开启压缩。
- 追求低延迟：减小 linger、控制 batch、减少同步等待。
- 追求可靠性：acks=all、开启幂等、合理 retries。

### Consumer 调优

重点参数：

- fetch.min.bytes
- fetch.max.bytes
- max.partition.fetch.bytes
- max.poll.records
- session.timeout.ms
- heartbeat.interval.ms
- max.poll.interval.ms

调优方向：

- 提高吞吐：批量处理、增加 partition、增加 consumer。
- 降低延迟：减少批量等待，优化业务处理耗时。
- 稳定消费：避免处理时间超过 poll 间隔，必要时拆分慢任务。

### 监控指标

必须监控：

- consumer lag
- producer send rate
- producer error rate
- request latency
- bytes in / bytes out
- under replicated partitions
- offline partitions
- broker disk usage
- JVM GC
- consumer 处理耗时
- dead letter topic 消息数量

### 必做实验

1. 写一个 Go benchmark，比较单条发送与批量发送。
2. 压测 100 万条消息，记录吞吐与 P95 延迟。
3. 故意让 consumer 变慢，观察 lag 增长。
4. 增加 consumer 数量，观察 lag 下降。
5. 增加 consumer 数量超过 partition 数量，观察吞吐是否继续提升。

## 第 6 阶段：生产运维与架构设计

时间建议：1 到 2 周。

### 运维基础

- topic 规划：
  - partition 数量
  - replication factor
  - retention
  - cleanup.policy
- 容量规划：
  - 消息大小
  - 峰值 QPS
  - 保留时间
  - 副本数量
  - 磁盘容量
- 安全：
  - TLS
  - SASL
  - ACL
- 故障处理：
  - broker 宕机
  - partition leader 切换
  - consumer lag 飙升
  - 磁盘打满
  - 大消息导致性能下降

### 架构设计能力

要能设计这些场景：

- 订单创建后异步通知多个下游服务。
- 用户行为日志写入 Kafka，再进入 ClickHouse 或 Elasticsearch。
- 使用 outbox pattern 保证数据库写入与事件发布的最终一致性。
- 使用 Kafka 做服务间事件驱动通信。
- 使用 Kafka Connect 做数据同步。
- 对热点 key 做分区策略优化。
- 为关键事件设计 schema 演进策略。

### 重点模式

- Outbox Pattern
- Retry Topic
- Dead Letter Topic
- Compacted Topic
- Event Sourcing 基础
- CQRS 基础
- Saga 与最终一致性

## 第 7 阶段：综合实战项目

时间建议：2 周。

### 项目：电商事件驱动后端

目标：做一个简化版电商后端，重点不是页面，而是 Kafka 驱动的后端链路。

### 服务划分

- `order-service`
  - 创建订单
  - 发布 `order.created`
- `inventory-service`
  - 消费 `order.created`
  - 扣减库存
  - 发布 `inventory.deducted` 或 `inventory.failed`
- `payment-service`
  - 模拟支付成功
  - 发布 `payment.succeeded`
- `notification-service`
  - 消费订单、支付、库存事件
  - 发送通知日志
- `event-worker`
  - 处理 retry topic 和 dead letter topic

### 技术要求

- Go + Kafka
- Docker Compose
- PostgreSQL 或 MySQL
- Redis 可选，用于幂等去重或缓存
- Prometheus metrics
- structured logging
- graceful shutdown
- integration tests

### 验收标准

- 可以一键启动全部依赖。
- 创建订单后能在 Kafka 中看到事件流转。
- 消费失败会进入 retry topic。
- 多次失败会进入 dead letter topic。
- 重复消息不会导致重复扣库存。
- consumer 重启后能从正确 offset 继续消费。
- 能在日志中定位任意一条消息的 topic、partition、offset、event_id。
- 有 README 说明架构、启动方式、关键配置和测试方式。

## 每周计划

### 第 1 周：跑起来并讲清楚

- 搭建本地 Kafka。
- 熟悉 topic、partition、offset、consumer group。
- 完成命令行生产与消费实验。
- 输出一篇笔记：《Kafka 为什么适合做事件流平台》。

### 第 2 周：吃透生产与消费

- 学 producer 参数。
- 学 consumer 参数。
- 做 key 分区、手动提交 offset、rebalance 实验。
- 输出一篇笔记：《Kafka 消息丢失和重复消费分别怎么发生》。

### 第 3 到 4 周：Go 客户端实战

- 选择一个 Go Kafka 客户端。
- 封装 producer 和 consumer。
- 完成订单事件异步处理小项目。
- 加入 retry topic 和 dead letter topic。

### 第 5 周：可靠性

- 做幂等消费。
- 做业务唯一键去重。
- 学事务和 exactly once。
- 实现 outbox pattern 的简化版本。

### 第 6 周：性能与监控

- 做压测。
- 对比不同 producer 参数。
- 监控 consumer lag。
- 加入 Prometheus 指标。

### 第 7 到 8 周：生产设计

- 学容量规划。
- 学安全配置。
- 学故障排查。
- 完成电商事件驱动后端项目。

### 第 9 到 12 周：进阶强化

- 学 Kafka Connect。
- 学 Schema Registry 或 Protobuf/Avro schema 管理。
- 学 Kafka Streams 的思想，Go 后端只需理解其模型即可。
- 阅读公司级 Kafka 事故复盘或技术博客。
- 把项目整理成作品集。

## Go 后端 Kafka 面试重点

### 高频问题

- Kafka 为什么快？
- Kafka 如何保证消息不丢？
- Kafka 如何处理重复消费？
- Kafka 如何保证顺序？
- partition 数量如何设计？
- consumer group rebalance 是什么？
- consumer lag 太高怎么排查？
- Kafka 的 ack 机制是什么？
- ISR 是什么？
- HW 和 LEO 是什么？
- log compaction 和 retention 有什么区别？
- retry topic 和 dead letter topic 怎么设计？
- Kafka 事务解决了什么问题？
- Kafka exactly once 的适用边界是什么？
- Go consumer 如何优雅退出？
- Go 服务中如何做 Kafka 消息幂等？

### 回答思路

不要只背配置名，要按这条线回答：

1. 业务目标是什么：吞吐、低延迟、可靠性、顺序性。
2. Kafka 提供了什么机制。
3. Go 后端代码里要怎么配合。
4. 还剩什么风险。
5. 如何监控和补救。

## 推荐资料

- Apache Kafka 官方文档：<https://kafka.apache.org/documentation/>
- Apache Kafka Quick Start：<https://kafka.apache.org/37/getting-started/quickstart/>
- Apache Kafka Monitoring：<https://kafka.apache.org/41/operations/monitoring/>
- Apache Kafka APIs：<https://kafka.apache.org/42/apis/>
- Confluent Go Client：<https://docs.confluent.io/kafka-clients/go/current/overview.html>
- confluent-kafka-go API：<https://docs.confluent.io/platform/current/clients/confluent-kafka-go/index.html>
- franz-go：<https://github.com/twmb/franz-go>
- Sarama：<https://github.com/IBM/sarama>

## 最小实战清单

学完后，至少应该在 GitHub 或本地作品集中留下这些东西：

- `docker-compose.yml`：本地 Kafka 环境。
- `producer.go`：支持可靠发送、日志、错误处理。
- `consumer.go`：支持消费者组、手动提交 offset、优雅关闭。
- `retry.go`：失败重试与死信 topic。
- `metrics.go`：消费 lag、处理耗时、错误数。
- `README.md`：说明架构图、启动方式、关键参数。
- `integration_test.go`：验证生产、消费、失败重试、幂等。

## 学习提醒

Kafka 学习最容易卡在两个地方：

1. 只学概念，不写消费者处理失败、重复消费、优雅退出这些真实代码。
2. 只会调 API，不理解 partition、offset、rebalance、ack、ISR 的底层含义。

最佳路径是：每学一个概念，立刻写一个 Go 实验验证它。Kafka 是工程味很重的技术，真正的理解来自“跑起来、弄坏它、再修好它”。
