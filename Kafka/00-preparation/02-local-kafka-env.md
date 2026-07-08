# 02 搭建本地 Kafka 环境

本节目标：使用 Docker Compose 启动一个单节点 Kafka，作为后续所有实验的基础环境。

## 为什么先用单节点

初学阶段最重要的是把概念跑通：

- topic 如何创建。
- producer 如何写入。
- consumer 如何读取。
- offset 如何变化。
- consumer group 如何分配 partition。

单节点足够完成这些实验。等你理解核心概念后，再学习多 broker、副本、故障恢复。

## 准备 Docker Compose

在项目根目录创建一个实验目录，例如：

```text
kafka-labs/
  docker-compose.yml
```

示例配置：

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

说明：

- `KAFKA_PROCESS_ROLES=broker,controller` 表示使用 KRaft 模式。
- `9092` 是客户端访问 Kafka 的端口。
- `9093` 是 controller 内部使用的端口。
- 本地学习用单节点，所以内部 topic 的副本因子设置为 `1`。

## 启动 Kafka

进入 `kafka-labs` 目录：

```powershell
docker compose up -d
docker ps
docker logs kafka
```

看到 Kafka 正常启动后，继续执行后续命令。

## 常见问题

### 端口 9092 被占用

检查本机是否已有 Kafka 或其他服务占用端口。

```powershell
netstat -ano | findstr :9092
```

可以把 Compose 中左侧端口改成其他值，例如：

```yaml
ports:
  - "19092:9092"
```

但这时 `KAFKA_ADVERTISED_LISTENERS` 也要对应修改。

### Go 程序连接不上 Kafka

重点检查：

- Go 程序是不是运行在宿主机。
- Kafka 暴露给宿主机的地址是不是 `localhost:9092`。
- 容器日志中有没有 listener 相关错误。

Kafka 的 advertised listener 非常关键。客户端最终会根据 broker 返回的 advertised 地址建立连接。如果这里填错，容器能启动，客户端也可能连不上。

### Windows 上路径或换行问题

尽量把实验文件放在普通英文目录下。Kafka 本身不怕中文路径，但许多命令行工具、脚本、终端编码会让新手多踩一些无关的坑。

## 本节练习

1. 启动 Kafka 容器。
2. 查看 Kafka 日志，确认没有明显错误。
3. 停止并重新启动容器，确认数据目录通过 volume 持久化。

