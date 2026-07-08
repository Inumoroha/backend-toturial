# 2. 使用 Kafka 命令行工具

本节目标：掌握 Kafka 初学阶段最常用的命令行工具，包括创建 topic、查看 topic、发送消息、消费消息和查看 consumer group。

学习 Kafka 时，不要一开始就写 Go 客户端。先用命令行工具把 Kafka 的行为看清楚，后面写 Go 代码时才知道客户端 API 背后发生了什么。

---

## 一、前置条件

开始之前，请确认：

- Kafka 容器已经启动。
- 能执行 `docker exec -it kafka bash` 进入容器。
- 已经读过 [1_使用Docker启动Kafka.md](1_使用Docker启动Kafka.md)。

进入容器：

```powershell
docker exec -it kafka bash
```

查看 Kafka 命令：

```bash
ls /opt/kafka/bin | grep kafka-
```

---

## 二、Kafka 常用命令行工具

初学阶段最常用的是这几个：

| 工具 | 用途 |
| --- | --- |
| `kafka-topics.sh` | 创建、查看、修改 topic |
| `kafka-console-producer.sh` | 从终端输入消息并发送到 Kafka |
| `kafka-console-consumer.sh` | 从 Kafka 读取消息并打印到终端 |
| `kafka-consumer-groups.sh` | 查看 consumer group、offset 和 lag |
| `kafka-configs.sh` | 查看或修改 topic/broker 配置 |

这一节先掌握前四个。

---

## 三、创建第一个 Topic

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic quickstart.orders \
  --partitions 3 \
  --replication-factor 1
```

参数解释：

- `--bootstrap-server localhost:9092`：连接 Kafka 的入口地址。
- `--create`：创建 topic。
- `--if-not-exists`：如果 topic 已存在，不报错。
- `--topic quickstart.orders`：topic 名称。
- `--partitions 3`：创建 3 个 partition。
- `--replication-factor 1`：每个 partition 只有 1 个副本。

本地单节点只能使用 `--replication-factor 1`。如果设置为 3，会失败，因为没有 3 个 broker。

---

## 四、查看 Topic 列表

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

预期能看到：

```text
quickstart.orders
tutorial.healthcheck
```

如果看不到 `quickstart.orders`，说明上一步创建可能失败。回去查看命令输出和 Kafka 日志。

---

## 五、查看 Topic 详情

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic quickstart.orders
```

你会看到类似：

```text
Topic: quickstart.orders  PartitionCount: 3  ReplicationFactor: 1
Topic: quickstart.orders  Partition: 0  Leader: 1  Replicas: 1  Isr: 1
Topic: quickstart.orders  Partition: 1  Leader: 1  Replicas: 1  Isr: 1
Topic: quickstart.orders  Partition: 2  Leader: 1  Replicas: 1  Isr: 1
```

字段解释：

- `PartitionCount`：分区数量。
- `ReplicationFactor`：副本数量。
- `Partition`：分区编号。
- `Leader`：当前负责读写的 broker。
- `Replicas`：这个 partition 的副本在哪些 broker 上。
- `Isr`：当前同步中的副本。

本地单节点时，Leader、Replicas、Isr 通常都是 `1`。

---

## 六、发送消息

启动 producer：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic quickstart.orders
```

终端会等待你输入。逐行输入：

```text
order-1 created
order-2 created
order-3 created
```

每按一次回车，就会发送一条消息。

退出 producer：

```text
Ctrl+C
```

---

## 七、消费消息

启动 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic quickstart.orders \
  --from-beginning
```

预期能看到：

```text
order-1 created
order-2 created
order-3 created
```

`--from-beginning` 表示从 topic 中当前可用的最早消息开始读。

如果不加这个参数，consumer 默认只会读取它启动之后的新消息。

---

## 八、使用 Consumer Group 消费

真实后端服务通常不会用匿名 consumer，而是使用 consumer group。

启动一个带 group 的 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic quickstart.orders \
  --group order-service \
  --from-beginning
```

这里的 `order-service` 可以理解为一个服务名。

同一个 topic 可以被多个 group 独立消费。例如：

```text
order-service
notification-service
analytics-service
```

它们各自有自己的消费进度。

---

## 九、查看 Consumer Group

查看 group 列表：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list
```

查看 `order-service` 的消费进度：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group order-service
```

可能看到：

```text
GROUP          TOPIC              PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
order-service  quickstart.orders  0          1               1               0
order-service  quickstart.orders  1          1               1               0
order-service  quickstart.orders  2          1               1               0
```

字段解释：

- `CURRENT-OFFSET`：当前 group 已经提交到的位置。
- `LOG-END-OFFSET`：partition 当前最新位置。
- `LAG`：还有多少消息没消费。

`LAG = LOG-END-OFFSET - CURRENT-OFFSET`。

---

## 十、打印 Key、Partition、Offset

为了观察消息进入哪个 partition，可以启动 consumer 时加属性：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic quickstart.orders \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true
```

如果消息没有 key，key 位置可能显示为 `null`。

后面 producer 章节会详细讲 key 如何影响 partition。

---

## 十一、带 Key 发送消息

启动 producer：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic quickstart.orders \
  --property parse.key=true \
  --property key.separator=:
```

输入：

```text
order-1001:created
order-1001:paid
order-1002:created
```

这里冒号左边是 key，右边是 value。

对于订单事件，常用 `order_id` 做 key。这样同一个订单的事件通常会进入同一个 partition，便于保证单个订单维度的顺序。

---

## 十二、常见问题

### 1. consumer 没读到历史消息

检查是否加了：

```bash
--from-beginning
```

如果使用了 consumer group，还要检查这个 group 是否已经消费过。

可以换一个新 group：

```bash
--group order-service-v2
```

### 2. 创建 topic 提示已经存在

加上：

```bash
--if-not-exists
```

### 3. group lag 一直是 0

可能原因：

- 消息已经被消费完。
- 当前 group 没有订阅这个 topic。
- 你查看错了 group 名。

### 4. 消息顺序和输入顺序不完全一样

如果 topic 有多个 partition，全局顺序不保证。

Kafka 只保证：

```text
同一个 partition 内的消息有序
```

---

## 十三、Go 后端工程视角

这些命令行工具对应到 Go 项目中就是：

| 命令行动作 | Go 后端代码 |
| --- | --- |
| 创建 topic | 运维脚本或 admin client |
| console producer | producer wrapper |
| console consumer | consumer group worker |
| 查看 group lag | 监控系统或排障命令 |

真实项目中，Go 服务不会每次启动都随意创建 topic。topic 通常由：

- 运维脚本。
- Terraform。
- Kubernetes Job。
- 管理平台。
- 初始化脚本。

来统一创建。

---

## 十四、本节练习

1. 创建 `quickstart.users` topic，设置 2 个 partition。
2. 向 `quickstart.users` 发送 5 条无 key 消息。
3. 使用 `--from-beginning` 消费它们。
4. 使用 `--group user-service` 再消费一次。
5. 查看 `user-service` 的 lag。
6. 再发送 3 条消息。
7. 查看 lag 是否增长。
8. 启动 consumer 消费，观察 lag 是否下降。
9. 使用 key 发送 `user-1:registered`、`user-1:profile_updated`。
10. 观察它们是否进入同一个 partition。

---

## 十五、本节小结

- `kafka-topics.sh` 用于管理 topic。
- `kafka-console-producer.sh` 可以从终端发送消息。
- `kafka-console-consumer.sh` 可以从 Kafka 读取消息。
- `kafka-consumer-groups.sh` 可以查看 group offset 和 lag。
- `--from-beginning` 只影响从哪里开始读，不会改变 Kafka 中的消息。
- consumer group 有自己的消费进度。
- offset 是 consumer group 在 partition 上的进度。
- 同一个 partition 内有序，多 partition 不保证全局有序。

---

## 十六、命令练习清单

请不看正文独立完成：

```text
list topics
describe topic
produce 3 messages
consume from beginning
describe group lag
```

能完成这五步，说明 Kafka CLI 已经具备基本熟练度。

---

## 十七、命令笔记建议

把每条命令记录成：

```text
命令：
用途：
关键参数：
常见错误：
```

后面排查 Kafka 问题时，这份命令笔记会非常有用。
