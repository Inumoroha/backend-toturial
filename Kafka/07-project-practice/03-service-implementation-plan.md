# 03 服务实现计划

本节目标：把项目拆成可以逐步完成的开发任务。

## 第一步：基础设施

准备：

- `docker-compose.yml`
- Kafka。
- PostgreSQL 或 MySQL。
- topic 创建脚本。
- Go module。
- 日志库。
- 配置加载。

验收：

- 一条命令启动依赖。
- 能用命令行创建和查看 topic。
- Go 程序能连接数据库和 Kafka。

## 第二步：Kafka 基础封装

实现：

- `internal/kafka/message.go`
- `internal/kafka/producer.go`
- `internal/kafka/consumer.go`
- `internal/kafka/retry.go`
- `internal/kafka/dlq.go`

验收：

- producer 能发送测试事件。
- consumer 能加入 group 并消费。
- 处理成功后手动提交 offset。
- 处理失败能写 retry 或 DLQ。

## 第三步：订单服务

实现：

- 创建订单 API。
- 订单表。
- outbox_events 表。
- 创建订单时写订单和 outbox。
- outbox worker 发布 `order.created`。

验收：

- 调用 API 后数据库有订单。
- outbox 有事件。
- worker 发布 Kafka 成功后 outbox 标记 sent。

## 第四步：库存服务

实现：

- 库存表。
- processed_events 去重表。
- 消费 `order.created`。
- 幂等扣减库存。
- 发布 `inventory.deducted` 或 `inventory.failed`。

验收：

- 重复消费同一 `event_id` 不会重复扣库存。
- 库存不足时发布失败事件。
- 数据库临时错误进入 retry topic。

## 第五步：支付与通知

实现：

- payment-service 模拟支付结果。
- notification-service 消费多个 topic。
- 通知记录表。

验收：

- 支付成功事件能被通知服务消费。
- 通知失败能重试。
- DLQ 有明确错误信息。

## 第六步：可观测性

实现：

- Prometheus metrics。
- structured logging。
- health check。
- lag 查看脚本。

验收：

- 能看到处理成功数。
- 能看到处理失败数。
- 能看到 handler 耗时。
- 日志可按 event_id 追踪。

## 本节练习

1. 把这些步骤写进项目 README 的任务列表。
2. 先实现第一条链路，不要同时开多个服务。
3. 为每一步写验收命令。
4. 思考：哪些模块需要单元测试，哪些需要集成测试？

