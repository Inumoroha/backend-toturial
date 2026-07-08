# 4. Prometheus 指标设计

本节目标：为 Go Kafka producer 和 consumer 设计 Prometheus 指标，能观察吞吐、延迟、错误、lag、retry 和 DLQ。

---

## 一、为什么需要指标

没有指标时，你只能靠日志猜：

```text
是不是消费慢？
是不是 Kafka 慢？
是不是数据库慢？
是不是某条坏消息卡住？
```

指标让你能量化问题。

---

## 二、Producer 指标

发送总数：

```text
kafka_producer_messages_total{topic,status}
```

发送耗时：

```text
kafka_producer_publish_duration_seconds{topic}
```

发送失败：

```text
kafka_producer_errors_total{topic,error_type}
```

---

## 三、Consumer 指标

处理总数：

```text
kafka_consumer_messages_total{topic,group,handler,status}
```

状态：

```text
success
retry
dlq
failed
```

处理耗时：

```text
kafka_consumer_handle_duration_seconds{topic,group,handler}
```

错误数：

```text
kafka_consumer_errors_total{topic,group,handler,error_type}
```

---

## 四、Lag 指标

```text
kafka_consumer_lag{topic,group,partition}
```

一定要包含 partition 标签。

只看 group 总 lag 容易掩盖热点 partition。

---

## 五、Retry 和 DLQ 指标

Retry：

```text
kafka_consumer_retry_messages_total{source_topic,retry_topic,handler}
```

DLQ：

```text
kafka_consumer_dlq_messages_total{source_topic,dlq_topic,handler}
```

关键业务 DLQ 新增就应该告警。

---

## 六、日志和指标配合

指标告诉你：

```text
哪里异常
```

日志告诉你：

```text
哪条消息异常
```

日志字段：

```text
topic partition offset key event_id group handler duration_ms error
```

---

## 七、告警规则示例

Lag：

```text
inventory-service lag > 1000 持续 5 分钟
```

DLQ：

```text
order.created.dlq 5 分钟内有新增
```

错误率：

```text
consumer error rate > 5% 持续 5 分钟
```

P99：

```text
handler P99 > 2s 持续 10 分钟
```

---

## 八、常见误区

- 只看日志不做指标。
- 只看平均耗时不看 P95/P99。
- lag 不带 partition 标签。
- DLQ 不告警。
- producer 失败没有指标。

---

## 九、本节练习

1. 为 producer 设计 3 个指标。
2. 为 consumer 设计 5 个指标。
3. 设计 lag 告警。
4. 设计 DLQ 告警。
5. 写出处理一条消息时的日志字段。

---

## 十、本节小结

- Kafka 链路必须可观测。
- Producer 要监控成功、失败、耗时。
- Consumer 要监控处理数、错误、耗时、lag。
- Lag 要按 partition 看。
- DLQ 必须告警。
- 日志和指标要配合使用。

---

## 十一、Go 指标定义示例

使用 Prometheus Go client 时，常见定义：

```go
var consumerMessages = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "kafka_consumer_messages_total",
        Help: "Total Kafka consumer handled messages.",
    },
    []string{"topic", "group", "handler", "status"},
)

var consumerDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "kafka_consumer_handle_duration_seconds",
        Help:    "Kafka consumer handler duration.",
        Buckets: prometheus.DefBuckets,
    },
    []string{"topic", "group", "handler"},
)
```

注册：

```go
prometheus.MustRegister(consumerMessages, consumerDuration)
```

---

## 十二、处理消息时记录指标

```go
started := time.Now()
status := "success"

err := handler.Handle(ctx, msg)
if err != nil {
    status = "failed"
}

consumerMessages.WithLabelValues(
    msg.Topic,
    groupID,
    handlerName,
    status,
).Inc()

consumerDuration.WithLabelValues(
    msg.Topic,
    groupID,
    handlerName,
).Observe(time.Since(started).Seconds())
```

如果进入 retry 或 DLQ，status 应该分别标记：

```text
retry
dlq
```

---

## 十三、Label 设计注意事项

Prometheus label 不能无限发散。

不要把下面字段作为 label：

```text
event_id
order_id
user_id
offset
```

这些值基数太高，会把 Prometheus 打爆。

适合作为 label：

```text
topic
group
handler
status
error_type
partition
```

高基数字段放日志，不放指标 label。

---

## 十四、Lag 指标来源

Lag 可以通过：

- Kafka exporter。
- 客户端暴露。
- 自己定时查询 group offset。

生产中更常见使用 Kafka exporter 或平台已有监控。

如果自己实现，要注意：

```text
不要高频查询 Kafka group offset，避免增加集群压力
```

---

## 十五、Dashboard 建议

一个 Kafka consumer dashboard 至少包含：

- group 总 lag。
- partition lag Top N。
- 每秒处理数。
- 错误率。
- handler P95/P99。
- retry 数。
- DLQ 数。
- rebalance 次数。

Producer dashboard：

- 每秒发送数。
- 发送失败数。
- 发送 P95/P99。
- bytes out。
- buffer 或队列积压。

---

## 十六、告警示例

用自然语言描述即可：

```text
inventory-service 的 order.created lag 超过 10000，持续 10 分钟。
```

```text
order.created.dlq 在 5 分钟内新增超过 0。
```

```text
inventory-service handler P99 超过 2 秒，持续 10 分钟。
```

```text
outbox_events 中最老 PENDING 超过 5 分钟。
```

---

## 十七、日志与 Trace

指标告诉你“异常在哪里”，日志告诉你“是哪条消息异常”。

建议日志：

```text
level=error msg="handle failed" topic=order.created partition=1 offset=203 key=order_1001 event_id=evt_1 group=inventory-service error_type=database_timeout
```

如果有 trace 系统，headers 中传：

```text
trace_id
```

consumer 处理时把 trace_id 写入日志。

---

## 十八、本节补充练习

1. 设计一个 consumer dashboard。
2. 判断 `event_id` 能不能作为 Prometheus label，并说明原因。
3. 为 outbox worker 设计 3 个指标。
4. 为 retry topic 设计告警。
5. 写一条带完整字段的错误日志。

---

## 十九、Dashboard 分区建议

一个 Kafka consumer dashboard 可以分成：

```text
吞吐区：消费速率、成功数、失败数
延迟区：handler P95/P99
积压区：group lag、partition lag
失败区：retry 数量、DLQ 数量
依赖区：数据库耗时、外部接口耗时
```

面板分区清楚，排查时才不会在一堆曲线里迷路。

---

## 二十、Label 再提醒

不要把 `event_id`、`order_id` 作为 Prometheus label。

它们应该进入日志，用来精确定位单条消息；指标 label 应该保持低基数，例如 `topic`、`group`、`handler`、`result`。

---

## 二十一、指标验收

一个 consumer 至少暴露：

```text
处理总数
处理失败数
处理耗时
DLQ 数量
当前 lag
```

这些指标能覆盖吞吐、错误、延迟和积压四类问题。

---

## 二十二、最终检查

如果某个 label 的取值会随着订单数量增长，它就不应该放进 Prometheus label。

---

## 二十三、指标命名练习

为订单消费链路设计指标时，可以这样命名：

```text
order_consumer_messages_total
order_consumer_failures_total
order_consumer_handler_duration_seconds
order_consumer_dlq_total
order_consumer_lag
```

命名时保持三点：业务域清楚、单位清楚、计数器和耗时类型清楚。

后续写 dashboard 时，指标名越稳定，排障文档越容易复用。
