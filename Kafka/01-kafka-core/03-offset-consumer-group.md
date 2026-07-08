# 03 Offset 与 Consumer Group

本节目标：理解 Kafka 如何记录消费进度，以及为什么 consumer group 是后端开发必须掌握的核心概念。

## Offset 是什么

offset 是消息在 partition 内的位置。

每个 partition 都有自己的 offset 序列：

```text
partition-0: 0, 1, 2, 3, 4
partition-1: 0, 1, 2, 3, 4
partition-2: 0, 1, 2, 3, 4
```

注意：offset 只在 partition 内有意义。不存在一个全局递增的 topic offset。

## Consumer Group 的 offset

Kafka 记录的是某个 consumer group 在某个 partition 上消费到了哪里。

可以理解为：

```text
group: order-service
topic: order.created
partition: 0
committed offset: 100
```

这表示 `order-service` 这个消费者组下一次应该从 partition 0 的某个位置继续读。

## 为什么 offset 属于 group

同一个 topic 可以被多个业务独立消费：

```text
order.created
  -> order-service
  -> notification-service
  -> analytics-service
```

每个 group 都应该有自己的进度。

如果 offset 属于 topic，那么一个服务读完后，其他服务就读不到了，这不符合 Kafka 的事件流模型。

## 提交 offset 的含义

提交 offset 的本质是告诉 Kafka：

```text
这个 group 已经处理到这里了。
```

对 Go 后端来说，正确时机通常是：

1. 拉到消息。
2. 执行业务逻辑。
3. 业务处理成功。
4. 提交 offset。

如果第 2 步还没成功就提交 offset，consumer 崩溃后这条消息可能不会再被处理，造成消息丢失。

## 自动提交与手动提交

### 自动提交

客户端定期提交 offset。

优点：

- 使用简单。

缺点：

- 可能业务还没处理成功，offset 已经提交。
- 出错时不容易控制。

### 手动提交

业务处理成功后由代码提交 offset。

优点：

- 可靠性更可控。
- 适合生产项目。

缺点：

- 代码复杂度更高。
- 必须正确处理重试、退出、异常。

Go 后端生产项目通常优先选择手动提交。

## Consumer Lag

lag 表示消费者落后了多少消息。

简化理解：

```text
lag = log end offset - committed offset
```

lag 高可能说明：

- consumer 处理太慢。
- consumer 数量太少。
- partition 数量太少。
- 下游数据库或接口慢。
- consumer 发生错误一直重试。
- consumer group 频繁 rebalance。

## 查看 group offset

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group order-service
```

重点字段：

- `CURRENT-OFFSET`：当前提交的 offset。
- `LOG-END-OFFSET`：partition 最新位置。
- `LAG`：积压量。
- `CONSUMER-ID`：消费者实例。

## 本节练习

1. 创建一个 group 消费消息。
2. 发送 10 条消息。
3. 消费其中一部分后停止 consumer。
4. 用 `kafka-consumer-groups.sh --describe` 查看 offset 和 lag。
5. 重新启动同一个 group，观察是否从之前的位置继续消费。
6. 换一个新的 group，观察是否可以从头消费。

