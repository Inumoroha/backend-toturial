# 05 环境验证与排错清单

本节目标：确认你的本地 Kafka 环境真的可用，而不是“容器看起来启动了”。Kafka 学习最烦人的不是概念，而是 listener、端口、group offset 这些小问题让实验现象对不上。

## 验证顺序

按这个顺序检查，不要跳步：

1. Docker 是否运行。
2. Kafka 容器是否运行。
3. Kafka 日志是否无明显错误。
4. 宿主机能否访问 `localhost:9092`。
5. 容器内能否创建 topic。
6. producer 能否写入。
7. consumer 能否读取。
8. consumer group 能否记录 offset。

## 1. 检查 Docker

```powershell
docker version
docker compose version
docker ps
```

预期结果：

- 能显示 Docker 版本。
- `docker ps` 不报错。

如果失败：

- 确认 Docker Desktop 已启动。
- Windows 上确认 WSL2 或 Hyper-V 配置正常。

## 2. 启动 Kafka

在包含 `docker-compose.yml` 的目录执行：

```powershell
docker compose up -d
docker ps
```

预期看到名为 `kafka` 的容器，状态是 running。

## 3. 查看 Kafka 日志

```powershell
docker logs kafka --tail 100
```

正常日志通常会显示 broker 启动、controller 启动、listener 就绪。

需要警惕的关键词：

```text
ERROR
FATAL
Exception
Unable to bind
Connection refused
```

如果端口冲突，优先检查：

```powershell
netstat -ano | findstr :9092
```

## 4. 在容器内创建验证 topic

```powershell
docker exec -it kafka bash
```

进入容器后：

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

预期结果：

- 能看到 `PartitionCount: 1`。
- leader 不为 `none`。

## 5. 写入并读取验证消息

启动 producer：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic tutorial.healthcheck
```

输入：

```text
hello kafka
```

另开一个终端进入容器，启动 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic tutorial.healthcheck \
  --from-beginning \
  --timeout-ms 10000
```

预期结果：能看到 `hello kafka`。

## 6. 验证 Consumer Group

使用 group 消费：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic tutorial.healthcheck \
  --group tutorial-healthcheck-group \
  --from-beginning \
  --timeout-ms 10000
```

查看 group：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group tutorial-healthcheck-group
```

重点看：

- `CURRENT-OFFSET`
- `LOG-END-OFFSET`
- `LAG`

如果 `LAG` 是 0，说明当前 group 已经消费到最新位置。

## 常见错误与处理

### 能进入容器，但宿主机 Go 程序连不上

优先检查 `KAFKA_ADVERTISED_LISTENERS`。

本地 Go 程序运行在宿主机时，advertised listener 应该能被宿主机访问，例如：

```text
PLAINTEXT://localhost:9092
```

如果 advertised listener 写成容器内部地址，Go 程序可能拿到一个宿主机无法访问的 broker 地址。

### topic 创建成功，但 consumer 读不到历史消息

检查是否使用了 `--from-beginning`。

还要检查是否使用了旧 group。旧 group 如果已经提交过 offset，再次启动不会从头读。

可以换一个新 group：

```bash
--group tutorial-healthcheck-group-v2
```

### consumer group 查不到

可能原因：

- consumer 没有指定 `--group`。
- consumer 没有真正消费到消息。
- group 信息还没刷新。

## 本节验收

你需要能完成：

- 创建 `tutorial.healthcheck` topic。
- 发送 `hello kafka`。
- 用 consumer 读到消息。
- 用 consumer group 读到消息。
- 用 `kafka-consumer-groups.sh --describe` 看到 offset 和 lag。

