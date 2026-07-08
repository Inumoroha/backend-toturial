# 07 最终交付结构与 README 模板

本节目标：把项目整理成可以展示、复盘、面试讲解的作品。

## 最终目录结构

```text
ecommerce-events/
  README.md
  go.mod
  go.sum
  cmd/
    order-service/
      main.go
    inventory-service/
      main.go
    outbox-worker/
      main.go
  internal/
    kafka/
      config.go
      message.go
      producer.go
      consumer.go
      retry.go
      dlq.go
      metrics.go
    order/
      api.go
      model.go
      repository.go
      service.go
      event.go
    inventory/
      handler.go
      repository.go
      service.go
      event.go
    outbox/
      worker.go
      repository.go
    platform/
      config/
      logger/
      database/
      shutdown/
      metrics/
  deployments/
    docker-compose.yml
    topics.sh
  migrations/
    001_init.sql
  tests/
    integration/
      order_flow_test.go
      idempotency_test.go
      dlq_test.go
```

## README 模板

~~~markdown
# Ecommerce Events

一个使用 Go + Kafka 实现的电商事件驱动后端示例。

## 功能

- 创建订单
- Outbox 发布 order.created
- 库存服务消费订单事件
- 幂等扣减库存
- Retry Topic
- Dead Letter Topic
- Consumer Lag 与处理耗时指标

## 架构

放 mermaid 图。

## 技术栈

- Go
- Kafka
- PostgreSQL
- Docker Compose
- Prometheus

## 快速启动

```bash
docker compose -f deployments/docker-compose.yml up -d
bash deployments/topics.sh
go run ./cmd/outbox-worker
go run ./cmd/inventory-service
go run ./cmd/order-service
~~~

## 创建订单

```bash
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id":"user_88","items":[{"sku_id":"sku_1","quantity":2,"price":9900}]}'
```

## 验证 Kafka

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --from-beginning
```

## 可靠性设计

- Outbox 保证订单写库和事件发布最终一致。
- Consumer 使用 processed_events 做幂等。
- 可重试错误进入 retry topic。
- 不可重试错误进入 DLQ。
- 业务成功后才提交 offset。

## 测试

```bash
go test ./...
go test ./tests/integration/...
```

## 故障演示

- 重复消息不会重复扣库存。
- 数据库临时故障进入 retry topic。
- 非法消息进入 DLQ。
- Consumer 停止时 lag 增长，恢复后下降。
```

## 面试讲解顺序

讲项目时按这条线：

1. 为什么引入 Kafka：解耦、异步、削峰。
2. 为什么需要 outbox：避免订单写库成功但事件丢失。
3. 为什么需要幂等：Kafka 至少一次会带来重复消费。
4. 为什么需要 retry 和 DLQ：失败不能卡住主链路。
5. 如何监控：lag、错误率、处理耗时、DLQ 数量。
6. 如何扩展：增加 partition、consumer、优化 handler。

## 最终自查

- [ ] README 能让别人一键启动。
- [ ] 架构图能说明事件流。
- [ ] 代码里没有硬编码 broker 地址。
- [ ] 每条消息都有 event_id。
- [ ] consumer 手动提交 offset。
- [ ] 重复消息不会造成重复扣库存。
- [ ] DLQ 消息有原始上下文。
- [ ] 日志能按 event_id 串起链路。
- [ ] 有至少 3 个集成测试：正常链路、幂等、DLQ。
