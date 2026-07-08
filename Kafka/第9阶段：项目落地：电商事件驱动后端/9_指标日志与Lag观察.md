# 9. 指标、日志与 Lag 观察

本节目标：为项目加入结构化日志和基础指标，让 Kafka 链路可观察、可排查。

---

## 一、日志字段

每条消息处理日志包含：

```text
topic
partition
offset
key
event_id
group
handler
duration_ms
error
```

---

## 二、Producer 指标

```text
kafka_producer_messages_total
kafka_producer_errors_total
kafka_producer_duration_seconds
```

---

## 三、Consumer 指标

```text
kafka_consumer_messages_total
kafka_consumer_errors_total
kafka_consumer_duration_seconds
kafka_consumer_dlq_total
```

---

## 四、Lag 观察命令

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

---

## 五、告警

关键告警：

- inventory-service lag 持续升高。
- DLQ 有新增。
- handler P99 太高。
- outbox PENDING 时间过长。

---

## 六、验收

- 能通过 event_id 查到完整链路日志。
- 能查看 group lag。
- 能看到 DLQ 数量。
- 能定位失败 topic、partition、offset。

---

## 七、结构化日志示例

处理成功：

```text
level=info msg="kafka message handled" topic=order.created partition=1 offset=203 key=order_1001 event_id=evt_order_created_order_1001 group=inventory-service handler=OrderCreatedHandler duration_ms=35
```

处理失败：

```text
level=error msg="kafka message failed" topic=order.created partition=1 offset=204 key=order_1002 event_id=evt_order_created_order_1002 group=inventory-service error_type=database_timeout error="context deadline exceeded"
```

---

## 八、Outbox 指标

```text
outbox_events_pending_total
outbox_events_failed_total
outbox_oldest_pending_age_seconds
outbox_publish_errors_total
```

最重要：

```text
outbox_oldest_pending_age_seconds
```

它代表事件发布延迟。

---

## 九、Lag 验证步骤

停止 inventory-service。

创建多笔订单。

查看 lag：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

重启 inventory-service。

再次查看 lag。

预期：

```text
lag 先增长，再下降。
```

---

## 十、DLQ 验证

消费 DLQ：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created.dlq \
  --from-beginning
```

确认 DLQ 消息包含：

- original_topic。
- original_partition。
- original_offset。
- error_message。
- payload。

---

## 十一、README 中要写的排查命令

```bash
# 查看 topic
kafka-topics.sh --bootstrap-server localhost:9092 --list

# 查看 group lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group inventory-service

# 查看 DLQ
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic order.created.dlq --from-beginning
```

---

## 十二、本节练习

1. 为成功处理写一条结构化日志。
2. 为失败处理写一条结构化日志。
3. 停止 consumer 制造 lag。
4. 发送坏消息制造 DLQ。
5. 把排查命令写入项目 README。

---

## 十三、Go 结构化日志封装

Go 1.21 以后可以直接使用 `log/slog`。建议在 Kafka consumer 中统一封装日志字段：

```go
func logMessageHandled(logger *slog.Logger, msg kafka.ConsumedMessage, eventID string, duration time.Duration) {
    logger.Info("kafka message handled",
        "topic", msg.Topic,
        "partition", msg.Partition,
        "offset", msg.Offset,
        "key", string(msg.Key),
        "event_id", eventID,
        "group", "inventory-service",
        "handler", "OrderCreatedHandler",
        "duration_ms", duration.Milliseconds(),
    )
}
```

失败日志：

```go
func logMessageFailed(logger *slog.Logger, msg kafka.ConsumedMessage, eventID string, err error, duration time.Duration) {
    logger.Error("kafka message failed",
        "topic", msg.Topic,
        "partition", msg.Partition,
        "offset", msg.Offset,
        "key", string(msg.Key),
        "event_id", eventID,
        "group", "inventory-service",
        "handler", "OrderCreatedHandler",
        "duration_ms", duration.Milliseconds(),
        "error", err.Error(),
    )
}
```

日志不是为了好看，而是为了排障时能用一行信息定位消息。

---

## 十四、Prometheus 指标示例

如果项目使用 `prometheus/client_golang`，可以先定义这些指标：

```go
var consumerMessagesTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "kafka_consumer_messages_total",
        Help: "Total number of consumed Kafka messages.",
    },
    []string{"topic", "group", "handler", "result"},
)

var consumerDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "kafka_consumer_duration_seconds",
        Help:    "Kafka message handling duration.",
        Buckets: prometheus.DefBuckets,
    },
    []string{"topic", "group", "handler"},
)
```

处理成功：

```go
consumerMessagesTotal.WithLabelValues("order.created", "inventory-service", "OrderCreatedHandler", "success").Inc()
consumerDuration.WithLabelValues("order.created", "inventory-service", "OrderCreatedHandler").Observe(duration.Seconds())
```

处理失败：

```go
consumerMessagesTotal.WithLabelValues("order.created", "inventory-service", "OrderCreatedHandler", "error").Inc()
```

---

## 十五、Outbox 指标 SQL

`outbox_events_pending_total` 可以用 SQL 计算：

```sql
SELECT count(*)
FROM outbox_events
WHERE status = 'PENDING';
```

`outbox_oldest_pending_age_seconds`：

```sql
SELECT EXTRACT(EPOCH FROM now() - MIN(created_at)) AS age_seconds
FROM outbox_events
WHERE status = 'PENDING';
```

如果这个值持续升高，说明：

```text
outbox-worker 没启动。
Kafka 不可用。
worker 发送失败但没有恢复。
worker 吞吐低于订单创建速度。
```

---

## 十六、Lag 输出怎么看

`kafka-consumer-groups.sh --describe` 会输出类似：

```text
GROUP              TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
inventory-service  order.created  0          120             130             10
inventory-service  order.created  1          98              98              0
inventory-service  order.created  2          76              80              4
```

字段含义：

| 字段 | 含义 |
| --- | --- |
| CURRENT-OFFSET | consumer group 已提交的位置 |
| LOG-END-OFFSET | 分区最新位置 |
| LAG | 还没处理的消息数 |

总 lag：

```text
10 + 0 + 4 = 14
```

如果 lag 一直升高，说明消费速度跟不上生产速度，或者 consumer 出错后没有提交 offset。

---

## 十七、按 event_id 串联排查

每条事件都要有 `event_id`。排查时按这个顺序找：

```text
1. order-service 日志：是否创建订单并写 outbox。
2. outbox-worker 日志：是否发布到 Kafka。
3. Kafka topic：是否存在消息。
4. inventory-service 日志：是否消费成功。
5. processed_events：是否记录处理结果。
6. inventory 表：库存是否变化。
```

常用 SQL：

```sql
SELECT id, status, retry_count, created_at, sent_at
FROM outbox_events
WHERE id = 'evt_order_created_order_1001';

SELECT event_id, handler, processed_at
FROM processed_events
WHERE event_id = 'evt_order_created_order_1001';
```

这就是为什么事件 id 不能随便省。

---

## 十八、告警阈值建议

学习项目可以先写成 README 规则：

| 指标 | 告警条件 |
| --- | --- |
| consumer lag | 连续 5 分钟大于 1000 |
| DLQ 数量 | 5 分钟内新增大于 0 |
| outbox oldest age | 大于 60 秒 |
| consumer error rate | 5 分钟内错误率大于 1% |
| handler P99 | 大于 500ms |

这些数字不是固定答案。真正生产里要按业务流量和 SLO 调整。

---

## 十九、常见错误

### 1. 日志没有 event_id

没有 event_id，跨服务排查会非常痛苦。

### 2. 只看服务是否存活，不看 lag

consumer 进程活着，不代表它在正常消费。lag 才能说明积压情况。

### 3. DLQ 没有告警

DLQ 如果没人看，就只是另一个垃圾桶。只要 DLQ 新增，就应该至少记录告警。

### 4. 指标 label 放太多动态值

不要把 `order_id`、`event_id` 放进 Prometheus label，会导致高基数问题。它们应该进日志，不应该进指标 label。

---

## 二十、排障练习

做一次完整演练：

```text
1. 停止 inventory-service。
2. 连续创建 20 笔订单。
3. 查看 inventory-service lag。
4. 启动 inventory-service。
5. 观察 lag 下降。
6. 发送 bad-json。
7. 查看 DLQ 是否新增。
8. 用 event_id 查询 outbox 和 processed_events。
```

写下结论：

```text
lag 能说明是否积压。
DLQ 能暴露不可自动处理的坏消息。
event_id 能串起跨服务日志和数据库记录。
```
