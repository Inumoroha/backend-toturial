# 04 Go Kafka 项目结构

本节目标：为后续项目实战准备一个清晰的 Go 后端目录结构。

## 推荐目录

```text
ecommerce-events/
  cmd/
    order-service/
      main.go
    inventory-service/
      main.go
    notification-service/
      main.go
  internal/
    kafka/
      config.go
      producer.go
      consumer.go
      message.go
      retry.go
      dlq.go
      metrics.go
    order/
      model.go
      event.go
      handler.go
      repository.go
    inventory/
      model.go
      handler.go
      repository.go
    platform/
      logger/
      config/
      shutdown/
  deployments/
    docker-compose.yml
  migrations/
  README.md
```

## `internal/kafka`

这一层只处理 Kafka 通用能力：

- producer 封装。
- consumer group 封装。
- retry/DLQ。
- 消息结构。
- metrics。

不要把订单、库存等业务逻辑写进这里。

## 业务模块

例如 `internal/order`：

- 定义订单模型。
- 定义订单事件。
- 处理 HTTP 请求。
- 写数据库。
- 调用 producer 发布事件。

例如 `internal/inventory`：

- 定义库存模型。
- 消费订单事件。
- 扣减库存。
- 发布库存事件。

## 配置建议

用环境变量配置：

```text
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=order-service
KAFKA_GROUP_ID=inventory-service
KAFKA_SECURITY_PROTOCOL=PLAINTEXT
```

不要把 broker 地址硬编码在业务代码里。

## 测试分层

### 单元测试

测试 handler 业务逻辑。

Kafka producer 可以 mock。

### 集成测试

启动 Kafka 和数据库，验证：

- producer 能写入事件。
- consumer 能处理事件。
- 失败消息能进入 retry 或 DLQ。
- 重复消息不会重复扣库存。

### 手工实验

用命令行查看 topic、group、lag。

## 本节练习

1. 创建上述目录结构。
2. 写出 `internal/kafka/message.go` 的字段设计。
3. 为 `order.created` 定义 Go struct。
4. 思考：Kafka wrapper 和业务 handler 的边界应该在哪里？

