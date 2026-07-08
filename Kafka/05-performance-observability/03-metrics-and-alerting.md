# 03 指标与告警

Kafka 链路必须可观测。没有指标，你只能在用户反馈“系统慢了”之后猜。

## Go Consumer 必备指标

### 处理数量

```text
kafka_consumer_messages_total{topic,group,handler,status}
```

status 可以是：

- success
- retry
- dlq
- failed

### 处理耗时

```text
kafka_consumer_handle_duration_seconds{topic,group,handler}
```

关注：

- avg
- P95
- P99

### 错误数量

```text
kafka_consumer_errors_total{topic,group,handler,error_type}
```

### DLQ 数量

```text
kafka_dlq_messages_total{topic,source_topic,handler}
```

DLQ 一旦持续增长，必须告警。

## Consumer Lag

lag 是最重要的 Kafka 指标之一。

建议按维度查看：

- group
- topic
- partition

只看 group 总 lag 可能掩盖热点 partition。

## Producer 必备指标

- 发送成功数。
- 发送失败数。
- 发送耗时。
- 重试次数。
- 当前缓冲区大小。
- delivery report 错误。

## Broker 指标

生产环境要关注：

- bytes in/out。
- messages in。
- request latency。
- under replicated partitions。
- offline partitions。
- broker disk usage。
- network processor idle。
- request handler idle。
- JVM GC。

## 告警建议

### Lag 告警

不要只按绝对值告警，要结合业务。

例如：

- 普通日志 group lag 超过 100 万并持续 30 分钟。
- 订单 group lag 超过 1000 并持续 5 分钟。
- 某 partition lag 持续增长。

### DLQ 告警

DLQ 有新增就应该至少通知。

关键业务 DLQ 应该立即告警。

### Error Rate 告警

错误率突然升高通常意味着下游依赖异常或消息格式变化。

## 日志与 Trace

每条消息日志应包含：

- trace_id
- event_id
- topic
- partition
- offset
- key
- group
- handler
- attempt
- duration_ms

这样可以把 HTTP 请求、数据库记录、Kafka 消息、consumer 日志串起来。

## 本节练习

1. 为 consumer 设计 Prometheus 指标。
2. 写出订单消费者的日志字段。
3. 设计三条告警规则：lag、DLQ、错误率。
4. 思考：为什么只看平均耗时不够？

