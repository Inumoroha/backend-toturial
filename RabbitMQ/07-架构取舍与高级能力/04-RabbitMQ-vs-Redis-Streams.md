# 04. RabbitMQ vs Redis Streams

## 1. Redis Streams 是什么

Redis Streams 是 Redis 中的流数据结构。

它类似 append-only log，并支持：

- XADD 写入。
- XREAD 读取。
- consumer group。
- pending entries。
- XACK 确认。

它适合 Redis 技术栈中的轻量事件流和任务处理。

## 2. 一句话区别

RabbitMQ：

```text
专业消息代理，擅长业务路由、任务队列、ack、retry、DLQ 和权限治理。
```

Redis Streams：

```text
Redis 内置流结构，适合轻量消息流和已有 Redis 生态中的消费组处理。
```

## 3. 模型对比

RabbitMQ：

```text
Exchange -> Queue -> Consumer
```

Redis Streams：

```text
Stream -> Consumer Group -> Consumer
```

RabbitMQ 的强项是 exchange 路由。

Redis Streams 的强项是 Redis 内部数据结构和消费组。

## 4. 消费确认

RabbitMQ：

```text
manual ack
nack/reject
requeue
DLX/DLQ
```

Redis Streams：

```text
XREADGROUP
PEL
XACK
XAUTOCLAIM / XCLAIM
```

Redis Streams 中，消费组会记录 pending entries，消费者处理完成后 XACK。

两者都需要处理：

```text
消费者崩溃后的未确认消息。
```

## 5. 路由能力

RabbitMQ 有强路由：

- direct。
- fanout。
- topic。
- headers。

Redis Streams 没有 RabbitMQ 这种 exchange/binding 模型。

如果你需要复杂业务路由，RabbitMQ 更合适。

如果只是：

```text
一个 stream，多消费者组读取
```

Redis Streams 可能更简单。

## 6. 运维取舍

如果团队已经有高可用 Redis，并且消息场景轻量：

```text
Redis Streams 可以减少新中间件。
```

但如果业务消息很关键，且需要：

- 明确路由。
- DLQ。
- 权限隔离。
- 丰富管理 UI。
- 大量队列治理。

RabbitMQ 更专业。

## 7. 适合 Redis Streams 的场景

- 已有 Redis 基础设施。
- 轻量消息流。
- 简单任务分发。
- 实时事件缓冲。
- 小团队不想额外部署 RabbitMQ。
- 延迟和吞吐要求适中。

## 8. 不适合 Redis Streams 的场景

- 复杂路由。
- 多业务队列治理。
- 需要成熟 DLQ/retry 拓扑。
- 需要 RabbitMQ Management UI 这类管理体验。
- 消息系统是核心基础设施，需要独立治理。

## 9. RabbitMQ 和 Redis Streams 怎么选

选择 RabbitMQ：

```text
业务消息较重要
需要复杂路由
需要 retry/DLQ
需要清晰队列治理
需要独立消息中间件能力
```

选择 Redis Streams：

```text
已有 Redis
消息场景简单
流量不大
希望快速实现轻量队列
可以接受自己补足部分治理能力
```

## 10. 本节小结

RabbitMQ 更像完整消息系统。

Redis Streams 更像 Redis 中的流式数据结构。

选择时不要只看：

```text
Redis 已经有了，顺手用。
```

还要看：

```text
失败恢复、权限、监控、DLQ、路由、长期治理。
```

下一节对比 RabbitMQ 和数据库任务表。

