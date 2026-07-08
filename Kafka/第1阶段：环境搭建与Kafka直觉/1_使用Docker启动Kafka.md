# 1. 使用 Docker 启动 Kafka

本节目标：使用 Docker Compose 在本地启动一个 Kafka，理解最小 Kafka 环境中的端口、listener、数据持久化和日常管理命令。

推荐初学阶段使用 Docker 启动 Kafka，原因是它更容易隔离环境、重置数据和反复做实验。Kafka 比 PostgreSQL 更依赖网络地址配置，如果直接安装到操作系统里，初学时很容易把环境问题和 Kafka 概念混在一起。

---

## 一、前置条件

开始之前，需要确认：

- 已安装 Docker Desktop。
- Docker Desktop 正在运行。
- 终端中可以执行 `docker --version`。
- 终端中可以执行 `docker compose version`。
- 本机 `9092` 端口没有被其他 Kafka 服务占用。

检查 Docker：

```powershell
docker --version
docker compose version
docker ps
```

如果 `docker ps` 报错，说明 Docker Desktop 可能还没有启动。

---

## 二、为什么 Kafka 启动配置比 PostgreSQL 复杂

PostgreSQL 本地启动时，通常只需要：

```text
端口
用户名
密码
数据目录
```

Kafka 还需要特别关注：

```text
broker 对外告诉客户端的连接地址是什么。
broker 内部 controller 使用什么地址。
客户端从宿主机连接，还是从容器网络连接。
```

Kafka 客户端连接 broker 时，不只是连一次 `localhost:9092`。客户端会先连接 bootstrap server，然后 broker 会返回集群元数据，里面包含 broker 自己声明的地址。这个地址就是 advertised listener。

如果 advertised listener 配错，就会出现：

```text
容器启动成功。
topic 也许能创建。
但是 Go 程序连接不上 Kafka。
```

所以本节会特别解释 listener。

---

## 三、创建 Docker Compose 文件

在 Kafka 学习目录下创建：

```text
kafka-labs/
  docker-compose.yml
```

`docker-compose.yml` 内容：

```yaml
services:
  kafka:
    image: apache/kafka:latest
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  kafka-data:
```

---

## 四、配置逐项解释

### 1. 镜像

```yaml
image: apache/kafka:latest
```

这里使用 Apache Kafka 官方镜像。学习阶段可以用 `latest`，如果你希望教程长期稳定，也可以固定版本。

生产环境不建议随意使用 `latest`，因为版本变化可能带来行为差异。

### 2. 端口映射

```yaml
ports:
  - "9092:9092"
```

含义：

- 左边的 `9092` 是宿主机端口。
- 右边的 `9092` 是容器内 Kafka 端口。

Go 程序运行在宿主机时，会连接：

```text
localhost:9092
```

### 3. KRaft 模式

```yaml
KAFKA_PROCESS_ROLES: broker,controller
```

现在 Kafka 可以使用 KRaft 模式运行，不需要再额外启动 ZooKeeper。

这里让单个 Kafka 节点同时扮演：

- broker：负责读写消息。
- controller：负责集群元数据管理。

初学阶段使用单节点 KRaft 最简单。

### 4. listeners

```yaml
KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
```

表示 Kafka 在容器内监听：

- `9092`：给客户端连接。
- `9093`：给 controller 内部通信。

### 5. advertised listeners

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
```

这是最容易出错的配置。

它表示 Kafka 告诉客户端：

```text
如果你想连接我，请使用 localhost:9092
```

因为我们的 Go 程序和命令行客户端会运行在宿主机，所以这里写 `localhost:9092`。

如果你的客户端运行在另一个 Docker 容器里，就不能写 `localhost`，而要写 Kafka 容器在 Docker 网络中的服务名，例如：

```text
kafka:9092
```

### 6. 内部 topic 副本数

```yaml
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
```

单节点环境只能设置为 1。

生产环境通常会使用多个 broker，并把副本数设置为 3。

### 7. 数据持久化

```yaml
volumes:
  - kafka-data:/var/lib/kafka/data
```

Kafka 消息会写入磁盘。这里把数据放到 Docker volume 中。

这样容器删除后，只要 volume 没删，数据仍然可以保留。

---

## 五、启动 Kafka

进入 `kafka-labs` 目录：

```powershell
docker compose up -d
```

查看容器：

```powershell
docker ps
```

预期能看到：

```text
NAMES     STATUS
kafka     Up ...
```

查看日志：

```powershell
docker logs kafka --tail 100
```

如果没有明显 `ERROR` 或 `FATAL`，说明 Kafka 基本启动成功。

---

## 六、进入 Kafka 容器

```powershell
docker exec -it kafka bash
```

进入后可以查看 Kafka 脚本：

```bash
ls /opt/kafka/bin | grep kafka-
```

常用脚本包括：

- `kafka-topics.sh`
- `kafka-console-producer.sh`
- `kafka-console-consumer.sh`
- `kafka-consumer-groups.sh`
- `kafka-configs.sh`

如果这些脚本存在，说明后续实验环境具备了。

---

## 七、检查 Kafka 是否真的可用

创建一个健康检查 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic tutorial.healthcheck \
  --partitions 1 \
  --replication-factor 1
```

查看 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic tutorial.healthcheck
```

正常情况下你会看到类似：

```text
Topic: tutorial.healthcheck
PartitionCount: 1
ReplicationFactor: 1
```

这一步比单纯看到容器 running 更可靠。容器启动成功不等于 Kafka 客户端链路可用。

---

## 八、日常管理命令

停止 Kafka：

```powershell
docker compose stop
```

重新启动：

```powershell
docker compose start
```

查看日志：

```powershell
docker logs kafka -f
```

停止并删除容器：

```powershell
docker compose down
```

删除容器和 volume：

```powershell
docker compose down -v
```

注意：`down -v` 会删除 volume，也就是删除 Kafka 中已有的数据。做实验想从零开始时可以用，平时不要随手执行。

---

## 九、端口冲突处理

如果启动时报错：

```text
Bind for 0.0.0.0:9092 failed: port is already allocated
```

说明本机 `9092` 被占用。

Windows 查看：

```powershell
netstat -ano | findstr :9092
```

可以把端口改为：

```yaml
ports:
  - "19092:9092"
```

但要注意，`KAFKA_ADVERTISED_LISTENERS` 也要改：

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:19092
```

否则客户端拿到的仍然是错误地址。

---

## 十、Go 后端工程视角

在 Go 项目里，Kafka 地址一般不会写死在代码中，而是通过环境变量传入：

```text
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=order-service
```

本地开发、测试环境、生产环境的 broker 地址通常不同。

一个简单的配置结构可能是：

```go
type KafkaConfig struct {
    Brokers  []string
    ClientID string
}
```

从环境变量读取：

```go
brokers := strings.Split(os.Getenv("KAFKA_BROKERS"), ",")
```

为什么要这么做？

- 本地可以连 `localhost:9092`。
- Docker Compose 内部服务可以连 `kafka:9092`。
- 生产环境可以连真实 broker 列表。
- 测试时可以替换成测试集群。

---

## 十一、常见错误

### 1. 容器启动了，但创建 topic 失败

先看日志：

```powershell
docker logs kafka --tail 200
```

再确认命令里的 bootstrap server：

```text
localhost:9092
```

### 2. Go 程序连接失败

优先检查：

- Go 程序运行在哪里？
- `KAFKA_ADVERTISED_LISTENERS` 是不是它能访问的地址？
- 端口映射是否正确？
- 防火墙是否拦截？

### 3. 重启后 topic 不见了

可能是你执行了：

```powershell
docker compose down -v
```

它会删除 volume。

如果只是想重启，不要带 `-v`。

---

## 十二、本节练习

1. 使用 Docker Compose 启动 Kafka。
2. 进入 Kafka 容器。
3. 创建 `tutorial.healthcheck` topic。
4. 使用 `kafka-topics.sh --describe` 查看 topic。
5. 停止 Kafka，再启动 Kafka。
6. 确认 `tutorial.healthcheck` topic 仍然存在。
7. 执行 `docker compose down -v` 后再次启动，观察 topic 是否还存在。
8. 用自己的话解释 `KAFKA_ADVERTISED_LISTENERS` 的作用。

---

## 十三、本节小结

- Kafka 本地学习建议用 Docker Compose。
- Kafka 比 PostgreSQL 更依赖网络地址配置。
- `KAFKA_LISTENERS` 是 Kafka 监听什么地址。
- `KAFKA_ADVERTISED_LISTENERS` 是 Kafka 告诉客户端使用什么地址连接它。
- Go 程序在宿主机运行时，本地 advertised listener 通常写 `localhost:9092`。
- 单节点学习环境的副本数只能设置为 1。
- volume 决定 Kafka 数据是否能在容器重建后保留。
- 判断 Kafka 是否可用，不能只看容器状态，还要实际创建 topic。

