# 05 核心概念实验课

本节目标：用一组小实验把 topic、partition、offset、consumer group、retention 这些概念真正看见。

## 实验准备

进入 Kafka 容器：

```powershell
docker exec -it kafka bash
```

创建实验 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic lab.orders \
  --partitions 3 \
  --replication-factor 1
```

查看详情：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic lab.orders
```

预期：能看到 3 个 partition。

## 实验 1：同一个 Group 内的 Partition 分配

终端 A：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.orders \
  --group lab-order-service \
  --property print.partition=true \
  --property print.offset=true
```

终端 B 再启动一个相同 group 的 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.orders \
  --group lab-order-service \
  --property print.partition=true \
  --property print.offset=true
```

终端 C 启动 producer：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.orders \
  --property parse.key=true \
  --property key.separator=:
```

输入：

```text
order-1:created
order-2:created
order-3:created
order-4:created
```

观察：

- 两个 consumer 会分摊 3 个 partition。
- 同一个 partition 不会同时被同一 group 内两个 consumer 消费。

## 实验 2：Consumer 数量超过 Partition 数量

再启动第三、第四个相同 group consumer。

预期：

- 前三个 consumer 可能分别拿到 partition。
- 第四个 consumer 大概率空闲。

结论：

```text
同一个 consumer group 的最大并行消费能力 <= partition 数量
```

## 实验 3：不同 Group 独立消费

启动另一个 group：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.orders \
  --group lab-analytics-service \
  --from-beginning
```

预期：

- `lab-analytics-service` 可以独立消费。
- 它不受 `lab-order-service` 已消费进度影响。

结论：

```text
offset 属于 consumer group，不属于 topic 本身
```

## 实验 4：查看 Offset 与 Lag

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group lab-order-service
```

字段解释：

- `CURRENT-OFFSET`：group 已提交的消费位置。
- `LOG-END-OFFSET`：partition 当前最新位置。
- `LAG`：还落后多少条。

如果你停止 consumer，然后继续发送消息，`LOG-END-OFFSET` 会增长，`CURRENT-OFFSET` 不变，`LAG` 增长。

## 实验 5：Retention

创建短保留 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic lab.retention \
  --partitions 1 \
  --replication-factor 1 \
  --config retention.ms=60000
```

写入消息：

```bash
echo "will-expire" | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.retention
```

等待一段时间后再消费：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.retention \
  --from-beginning \
  --timeout-ms 10000
```

注意：

- retention 清理不是精确到毫秒立即删除。
- Kafka 会按日志段和清理周期处理。
- 本地实验中可能需要多等一会儿。

## 实验记录模板

```markdown
# 实验：Consumer 数量超过 Partition 数量

## 操作

## 观察

## 结论

## 和 Go 后端的关系
```

## 本节验收

你需要能解释这些观察：

- 为什么第四个 consumer 没有分到 partition？
- 为什么换一个 group 可以重新读到历史消息？
- 为什么停止 consumer 后 lag 会增长？
- 为什么 retention 到期不一定立刻看见消息删除？

