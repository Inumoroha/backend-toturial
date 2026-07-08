# 1. Broker、Controller 与 KRaft

本节目标：理解 Kafka 集群里的 broker、controller 和 KRaft 模式，知道单节点学习环境为什么可以不启动 ZooKeeper，也能用命令观察当前 broker 和 topic 元数据。

学习 Kafka 时，很多教程一上来就讲 topic、partition、producer、consumer。它们当然重要，但如果你不知道 broker 和 controller 是什么，后面看到 leader 选举、partition 分配、broker 宕机、ISR 变化，就会很容易迷路。

---

## 一、先建立整体图景

一个 Kafka 集群可以简单理解成：

```text
Kafka Cluster
  broker-1
  broker-2
  broker-3
```

broker 是 Kafka 服务器节点。

producer 写消息时，最终写入某个 broker 上的某个 partition leader。

consumer 读消息时，也会从 broker 拉取某个 partition 的数据。

controller 则负责集群元数据管理，例如：

- 哪些 broker 存活。
- topic 有哪些 partition。
- partition leader 在哪个 broker。
- broker 宕机后谁接替 leader。

---

## 二、Broker 是什么

Broker 是 Kafka 的服务端节点。

它负责：

- 接收 producer 写入。
- 持久化消息到磁盘。
- 响应 consumer 拉取。
- 保存 partition 副本。
- 和其他 broker 协作复制数据。
- 向客户端返回 topic 和 partition 元数据。

在本地单节点环境中，只有一个 broker。

在生产环境中，通常会有多个 broker。

例如：

```text
broker-1
broker-2
broker-3
```

一个 topic 的不同 partition 可以分散在不同 broker 上。

---

## 三、Controller 是什么

Controller 可以理解成 Kafka 集群的协调者。

它不负责普通消息读写的主路径，但负责管理元数据和协调变化。

例如：

```text
broker-2 挂了
controller 发现 broker-2 不可用
controller 为受影响的 partition 选择新的 leader
其他 broker 和客户端感知新的元数据
```

如果没有 controller，Kafka 集群就不知道：

- 哪个 broker 是活的。
- 哪个 partition 应该由谁负责。
- broker 变化后如何重新分配 leader。

---

## 四、ZooKeeper 模式和 KRaft 模式

早期 Kafka 使用 ZooKeeper 存储和协调元数据。

架构大致是：

```text
Kafka brokers <-> ZooKeeper
```

现在 Kafka 支持 KRaft 模式，不再需要单独启动 ZooKeeper。

KRaft 模式下，Kafka 自己使用内部的元数据日志来管理集群元数据。

本教程使用的 Docker Compose 是 KRaft 模式：

```yaml
KAFKA_PROCESS_ROLES: broker,controller
KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
```

含义是：

```text
这个 Kafka 节点同时作为 broker 和 controller。
controller quorum 中只有一个节点。
```

这适合本地学习。

生产环境中，controller quorum 通常不会只有一个节点。

---

## 五、为什么初学阶段不讲 ZooKeeper

原因很简单：

```text
你作为 Go 后端工程师，当前最需要掌握的是 Kafka 的读写、消费、可靠性和业务落地。
```

ZooKeeper 是历史架构的重要部分，但对新手来说容易增加环境复杂度。

你应该先理解：

- topic。
- partition。
- producer。
- consumer group。
- offset。
- retry。
- DLQ。
- outbox。

等这些都清楚后，再了解 Kafka 历史上的 ZooKeeper 模式会更自然。

---

## 六、查看当前 Kafka 节点信息

进入容器：

```powershell
docker exec -it kafka bash
```

查看 topic 列表：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

这条命令会连接 broker，向 broker 查询 topic 元数据。

查看某个 topic 详情：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic quickstart.orders
```

你会看到：

```text
Leader: 1
Replicas: 1
Isr: 1
```

在单节点环境中：

- broker id 是 `1`。
- leader 是 `1`。
- replicas 是 `1`。
- ISR 是 `1`。

这表示所有 partition 都在同一个 broker 上。

---

## 七、Broker、Topic、Partition 的关系

可以这样理解：

```text
broker 是机器或服务节点。
topic 是逻辑消息分类。
partition 是 topic 的物理分片。
partition 副本存放在 broker 上。
```

单节点：

```text
broker-1
  order.created partition-0
  order.created partition-1
  order.created partition-2
```

三节点：

```text
broker-1
  order.created partition-0 leader

broker-2
  order.created partition-1 leader

broker-3
  order.created partition-2 leader
```

生产环境中，Kafka 会把 partition 分布到多个 broker 上，从而提升吞吐和可用性。

---

## 八、为什么 Go 客户端只配置 bootstrap brokers 就够了

Go Kafka 客户端一般配置：

```text
localhost:9092
```

或生产环境：

```text
kafka-1:9092,kafka-2:9092,kafka-3:9092
```

这些地址叫 bootstrap brokers。

客户端并不是永远只连接这几个地址。它会：

1. 先连接 bootstrap broker。
2. 获取集群元数据。
3. 知道 topic 的 partition leader 在哪些 broker。
4. 后续直接和对应 broker 通信。

所以 bootstrap broker 的作用是：

```text
帮助客户端进入 Kafka 集群
```

不是：

```text
所有读写都只经过这个地址
```

这就是为什么 advertised listener 配错会导致客户端异常。客户端拿到的元数据里 broker 地址不可达，就无法继续工作。

---

## 九、模拟 Broker 不可用的直觉

本地单节点环境中，如果停止 Kafka：

```powershell
docker stop kafka
```

此时：

- producer 无法写入。
- consumer 无法读取。
- topic 元数据也无法查询。

重新启动：

```powershell
docker start kafka
```

如果 volume 没删除，topic 和消息仍然存在。

这说明：

```text
Kafka 消息是持久化到磁盘的。
```

但单节点没有高可用。

生产环境需要多个 broker 和副本，后面会讲 replica、leader、ISR。

---

## 十、Go 后端工程视角

在 Go 项目中，broker 信息通常写在配置里：

```go
type KafkaConfig struct {
    Brokers  []string
    ClientID string
}
```

环境变量：

```text
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=order-service
```

读取配置：

```go
brokers := strings.Split(os.Getenv("KAFKA_BROKERS"), ",")
```

生产环境通常会配置多个 broker：

```text
KAFKA_BROKERS=kafka-1:9092,kafka-2:9092,kafka-3:9092
```

为什么配置多个？

- 任意一个 broker 不可用时，客户端仍然可能通过其他 broker 获取元数据。
- 提升客户端启动和恢复的稳定性。

注意：

```text
配置多个 bootstrap broker 不等于消息写多份。
消息副本由 topic 的 replication factor 决定。
```

---

## 十一、常见误区

### 1. broker 就是 topic

不是。

broker 是 Kafka 服务节点，topic 是逻辑消息分类。

一个 broker 上可以存很多 topic 的很多 partition。

### 2. controller 负责转发所有消息

不是。

controller 主要负责元数据和协调，不是所有消息读写的中转站。

### 3. 配置多个 bootstrap broker 就等于高可用

不完全是。

真正的高可用还需要：

- 多 broker。
- topic replication factor 大于 1。
- 合理的 min ISR。
- producer 使用合适的 acks。

### 4. KRaft 只是本地开发模式

不是。

KRaft 是 Kafka 新的元数据管理模式。只是本教程在本地用单节点 KRaft 来降低学习成本。

---

## 十二、本节练习

1. 查看 `quickstart.orders` 的 topic 详情。
2. 记录每个 partition 的 leader、replicas、ISR。
3. 停止 Kafka 容器。
4. 尝试执行 `kafka-topics.sh --list`，观察报错。
5. 重新启动 Kafka。
6. 再次查看 topic 是否还存在。
7. 用自己的话解释 broker 和 controller 的区别。
8. 用自己的话解释 bootstrap broker 的作用。

---

## 十三、本节小结

- broker 是 Kafka 服务节点，负责消息读写和存储。
- controller 负责集群元数据管理和协调。
- KRaft 模式下 Kafka 不需要单独依赖 ZooKeeper。
- 本地学习可以使用单节点 broker/controller。
- topic 的 partition 副本存放在 broker 上。
- Go 客户端通过 bootstrap broker 获取集群元数据。
- advertised listener 配错会导致客户端拿到不可访问的 broker 地址。
- 多 broker、高副本、合理 acks 才能构成生产高可用基础。

---

## 十六、角色复盘

用一句话记住：

```text
broker 存数据，controller 管元数据，client 通过 bootstrap broker 找到集群。
```

KRaft 只是把过去 ZooKeeper 管理的元数据职责，放回 Kafka 自己的 quorum 中。

---

## 十七、排查提示

如果 Go 客户端能连上 bootstrap broker，但随后报无法连接其他 broker，优先检查：

```text
advertised.listeners
```

很多本地 Docker Kafka 连接问题都出在这里。

---

## 二十一、排障时先确认哪几件事

遇到 broker 连不上时，建议按固定顺序检查，而不是反复重启容器。

```text
1. 容器是否还在运行。
2. broker 日志里是否完成启动。
3. advertised.listeners 是否暴露给客户端所在网络。
4. 客户端使用的地址是否和 listener 匹配。
5. topic 命令能否连接 bootstrap server。
6. producer 和 consumer 是否使用同一个 bootstrap server。
```

这和排查 PostgreSQL 连接失败类似：先确认服务存在，再确认监听地址，再确认客户端连接方式。

Kafka 多了一层复杂性：broker 返回给客户端的地址必须也是客户端能访问的地址。
