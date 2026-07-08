# 5. Consumer Lag 观察与排查

本节目标：理解 consumer lag 的含义，掌握使用命令查看 lag，并能按步骤排查 lag 持续升高的问题。

Lag 是 Kafka 消费链路最重要的指标之一。Go 后端工程师不需要每天运维 Kafka 集群，但必须能看懂 lag，因为它直接反映消费者是否跟得上生产者。

---

## 一、Lag 是什么

Lag 表示 consumer group 落后多少消息。

简化公式：

```text
lag = log end offset - committed offset
```

例如：

```text
LOG-END-OFFSET = 1000
CURRENT-OFFSET = 900
LAG = 100
```

表示这个 group 在这个 partition 上还有 100 条消息没处理。

---

## 二、Lag 高一定是坏事吗

不一定。

短时间 lag 升高可能只是：

- 上游突发写入。
- consumer 正常排队处理。
- 批处理任务正在运行。

真正需要关注：

- lag 持续增长。
- lag 长时间不下降。
- 单个 partition lag 特别高。
- lag 高同时错误率高。
- 关键业务超过可接受延迟。

---

## 三、查看 Lag

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

关注字段：

- `TOPIC`
- `PARTITION`
- `CURRENT-OFFSET`
- `LOG-END-OFFSET`
- `LAG`
- `CONSUMER-ID`

不要只看总 lag。要看每个 partition。

如果只有一个 partition lag 高，可能是热点 key 或坏消息。

---

## 四、制造 Lag 实验

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage4.lag.demo \
  --partitions 3 \
  --replication-factor 1
```

发送消息：

```bash
for i in $(seq 1 100); do
  echo "order-$i:created-$i"
done | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage4.lag.demo \
  --property parse.key=true \
  --property key.separator=:
```

在 consumer 未启动时查看 group 可能还不存在。

启动 consumer group 消费一次：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage4.lag.demo \
  --group lag-demo-group \
  --from-beginning
```

停止后继续发送消息，再查看 lag。

---

## 五、Lag 排查第一步：写入是否增加

先问：

```text
producer 写入量是不是突然变大？
```

如果上游突然从每秒 100 条变成每秒 10000 条，consumer lag 增长是自然结果。

需要判断：

- 是否临时峰值。
- consumer 是否能追上。
- 是否需要扩容。

---

## 六、Lag 排查第二步：Consumer 是否报错

看 consumer 日志：

```text
json decode failed
database timeout
http request timeout
commit failed
retry topic publish failed
```

如果大量错误导致重试，消费速度会下降。

不可重试错误应该进入 DLQ，不要卡住主链路。

---

## 七、Lag 排查第三步：Handler 是否变慢

关键指标：

```text
handler duration P95/P99
```

如果 P99 很高，说明少数慢消息可能拖住 partition。

常见慢点：

- 数据库慢查询。
- 下游 HTTP 慢。
- 锁竞争。
- 外部接口限流。
- 单条消息业务太重。

---

## 八、Lag 排查第四步：是否热点 Partition

如果只有一个 partition lag 高，常见原因：

- 热点 key。
- 该 partition 有坏消息。
- 分配到该 partition 的 consumer 实例异常。

处理：

- 查该 partition 日志。
- 找到失败 offset。
- 分析 key。
- 必要时把坏消息写 DLQ。

---

## 九、Lag 排查第五步：Consumer 数量和 Partition 数

扩容 consumer 前先看：

```text
topic partition 数量
当前 consumer 数量
```

如果：

```text
partition=3
consumer=6
```

继续加 consumer 没用。

需要：

- 优化 handler。
- 增加 partition。
- 拆分 topic。
- 处理热点 key。

---

## 十、Go 后端监控建议

至少监控：

```text
kafka_consumer_messages_total
kafka_consumer_errors_total
kafka_consumer_handle_duration_seconds
kafka_consumer_lag
kafka_consumer_dlq_total
```

日志字段：

```text
topic
partition
offset
key
event_id
duration_ms
error_type
```

---

## 十一、常见误区

### 1. Lag 高就加 consumer

不一定有效。

先看 partition 数和瓶颈。

### 2. Lag 为 0 就一定没问题

不一定。

可能 consumer 根本没消费、group 错了、监控错了。

### 3. 只看 group 总 lag

不够。

要看 partition 维度。

### 4. Lag 是 Kafka broker 的问题

大多数时候是 consumer 或下游依赖问题。

---

## 十二、本节练习

1. 创建 topic 并发送 100 条消息。
2. 停止 consumer 后继续发送消息。
3. 查看 group lag。
4. 重启 consumer，观察 lag 下降。
5. 设计 lag 高的排查流程。
6. 思考：如果只有 partition-1 lag 高，优先查什么？

---

## 十三、本节小结

- lag 表示 group 落后多少消息。
- lag 要按 partition 观察。
- lag 短期升高不一定是事故。
- lag 持续升高需要排查写入量、错误率、handler 耗时、热点 partition、rebalance。
- 增加 consumer 不一定有效，受 partition 数限制。
- Go 后端必须记录处理耗时和错误类型。

---

## 十四、Lag 排查顺序

看到 lag 高，不要第一反应就是扩容。按顺序查：

1. producer 写入速度是否突然增加。
2. consumer 是否有错误日志。
3. handler P95/P99 是否升高。
4. 数据库慢查询是否增加。
5. 是否只有某个 partition lag 高。
6. 最近是否频繁 rebalance。
7. DLQ 或 retry 是否异常增长。

常用命令：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

---

## 十五、热点 Partition

如果只有一个 partition lag 很高，通常说明 key 分布不均。

例如大量消息都使用：

```text
key = default
```

这会让扩容 consumer 也无效，因为同一个 partition 不能被同 group 多个 consumer 同时消费。

---

## 十六、告警建议

学习项目可以先用简单规则：

```text
lag 连续 10 分钟增长。
单 partition lag 明显高于其他 partition。
DLQ 有新增。
handler P99 超过预期阈值。
```

告警要能指向动作，而不是只告诉你“有问题”。

---

## 十七、排查表

| 现象 | 可能原因 | 动作 |
| --- | --- | --- |
| 所有 partition lag 增长 | 写入变多或 consumer 慢 | 看写入速率和 handler 耗时 |
| 单 partition lag 高 | 热点 key | 查看 key 分布 |
| lag 增长且错误率高 | handler 失败 | 看日志和 DLQ |
| lag 增长且 rebalance 多 | consumer 不稳定 | 看退出、超时、心跳 |
| lag 不降但服务存活 | handler 卡住 | 看 goroutine、DB、外部接口 |

排查时尽量把“现象、证据、动作”写清楚。

---

## 十八、常用日志字段

```text
topic
partition
offset
group
key
event_id
handler
duration_ms
error_kind
```

没有这些字段，lag 高时很难定位到具体消息和处理逻辑。

---

## 十九、排障案例

```text
现象：总 lag 50000。
进一步查看：partition 3 lag 48000，其他 partition 很低。
判断：不是整体消费能力不足，而是单 partition 热点。
动作：检查 message key，发现大量消息 key=default。
修复：改为 order_id，并评估是否需要拆分热点业务。
```

排障要从总量下钻到 partition。

---

## 二十、最终验收

完成本节后，拿一段真实或模拟的 lag 输出，写出：

```text
总 lag：
最高 lag partition：
是否热点：
是否需要扩 consumer：
下一步动作：
```

如果只能说“lag 很高”，说明排查还停留在表面。

---

## 二十一、lag 排查报告模板

建议把一次 lag 排查写成这样：

```text
topic：
group：
总 lag：
lag 最大的 partition：
该 partition 最近写入速度：
consumer 实例数：
单条 handler 平均耗时：
是否发生 rebalance：
下游数据库或接口是否变慢：
采取的动作：
复测结果：
```

这个模板能强迫你从“现象”走到“原因”。真实生产环境里，lag 是结果，不是原因。
