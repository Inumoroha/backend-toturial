# 03 第一次用命令行操作 Kafka

本节目标：不用写代码，先用 Kafka 自带命令行工具完成 topic 创建、消息发送、消息消费。

## 进入 Kafka 容器

```powershell
docker exec -it kafka bash
```

后续命令在容器内执行。

## 查看命令行工具

Kafka 的脚本通常在 `/opt/kafka/bin` 或镜像预置路径中。可以先搜索：

```bash
ls /opt/kafka/bin | grep kafka-
```

常用脚本：

- `kafka-topics.sh`
- `kafka-console-producer.sh`
- `kafka-console-consumer.sh`
- `kafka-consumer-groups.sh`
- `kafka-configs.sh`

## 创建 topic

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic order.created \
  --partitions 3 \
  --replication-factor 1
```

解释：

- `--topic order.created`：topic 名称。
- `--partitions 3`：创建 3 个分区。
- `--replication-factor 1`：单节点环境只能设置 1。

## 查看 topic

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

查看详情：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order.created
```

重点观察：

- partition 数量。
- 每个 partition 的 leader。
- replicas。
- ISR。

## 发送消息

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created
```

然后输入：

```text
order-1 created
order-2 created
order-3 created
```

按 `Ctrl+C` 退出 producer。

## 消费消息

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --from-beginning
```

如果能看到刚才发送的消息，说明 Kafka 基础链路已经跑通。

## 使用 consumer group

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --group order-service \
  --from-beginning
```

查看消费者组：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list
```

查看消费者组详情：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group order-service
```

重点观察：

- `CURRENT-OFFSET`
- `LOG-END-OFFSET`
- `LAG`
- `CONSUMER-ID`
- `HOST`
- `CLIENT-ID`

## 本节要理解的关键点

同一个 topic 可以被多个 consumer group 独立消费。

例如：

- `order-service` 消费订单事件做库存扣减。
- `notification-service` 消费订单事件发通知。
- `analytics-service` 消费订单事件做统计。

它们互不影响，因为每个 group 有自己的 offset。

## 本节练习

1. 创建 `user.registered` topic，分区数设置为 2。
2. 向 `user.registered` 发送 5 条消息。
3. 用两个不同的 group 分别消费，观察它们是否都能从头读到消息。
4. 使用 `kafka-consumer-groups.sh --describe` 查看 lag。

