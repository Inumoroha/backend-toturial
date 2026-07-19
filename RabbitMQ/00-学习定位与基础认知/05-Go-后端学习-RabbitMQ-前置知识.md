# 05. Go 后端学习 RabbitMQ 前置知识

## 1. Go 基础能力

学习 RabbitMQ 之前，你不需要成为 Go 专家，但下面这些能力最好具备。

### 1.1 context

你需要理解：

- `context.WithTimeout`
- `context.WithCancel`
- 请求级别超时
- 进程退出时取消任务

RabbitMQ 发布消息、消费消息、优雅退出都会用到 context 思维。

示例理解：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

含义是：这个操作最多执行 5 秒，超时就取消。

### 1.2 goroutine 和 channel

消费者通常会涉及并发处理：

```text
一个消费者进程
  -> 多个 worker goroutine
  -> 并发处理消息
```

你需要知道：

- 如何启动 goroutine。
- 如何等待 goroutine 退出。
- 如何通过 channel 传递信号。
- 如何避免 goroutine 泄漏。

### 1.3 错误处理

消息系统里，错误处理非常关键。

你需要能区分：

- 临时错误：可以重试，例如网络抖动。
- 永久错误：不该重试，例如消息格式错误。
- 业务错误：需要根据业务决定 ack、nack 或进入死信。

### 1.4 优雅退出

消费者进程不能随便强杀。

你要能处理：

- 收到退出信号。
- 停止接收新消息。
- 等待正在处理的消息完成。
- 成功处理后 ack。
- 超时后退出并让消息重新投递。

## 2. 后端基础能力

### 2.1 HTTP 和 RPC

你要能判断：

- 哪些场景适合同步 HTTP/RPC。
- 哪些场景适合异步消息。

例如：

```text
查询用户详情 -> 通常适合同步 HTTP/RPC
发送欢迎邮件 -> 通常适合异步消息
```

### 2.2 数据库事务

RabbitMQ 经常和数据库一起使用。

你需要理解：

- 本地事务
- 唯一索引
- 状态流转
- 乐观锁
- 事务提交失败
- 事务成功但消息发送失败

后续学习 Outbox 时，这些基础非常重要。

### 2.3 幂等

幂等是消息消费最重要的业务能力之一。

你要能设计：

- 如何识别同一条消息。
- 如何防止重复扣库存。
- 如何防止重复发优惠券。
- 如何让重复消费直接返回成功。

常用工具：

- 数据库唯一索引。
- 已处理消息表。
- 业务状态机。
- Redis 去重，适合部分有时效的场景。

### 2.4 日志和链路追踪

异步系统排查问题需要日志。

建议每条消息都带：

- `message_id`
- `correlation_id`
- `event_type`
- `routing_key`
- `retry_count`
- `created_at`

日志至少记录：

- 什么时候发布消息。
- 发布是否成功。
- 哪个消费者收到消息。
- 处理是否成功。
- 是否 ack。
- 是否进入重试或死信。

## 3. Docker 基础

后面学习 RabbitMQ 会大量使用 Docker。

你至少要会：

```bash
docker version
docker ps
docker logs
docker stop
docker rm
```

以及 Docker Compose：

```bash
docker compose version
docker compose up -d
docker compose down
docker compose logs -f
```

后续 RabbitMQ 本地环境通常会用：

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:management
```

其中：

- `5672` 是 AMQP 协议端口，Go 程序连接它。
- `15672` 是管理后台端口，浏览器访问它。

## 4. Linux 和网络基础

不需要特别深，但至少理解：

- TCP 长连接。
- 连接超时。
- 心跳。
- 端口。
- DNS。
- 防火墙。
- 进程和信号。

RabbitMQ 客户端会维护长连接。生产环境中，频繁创建和关闭连接是很糟糕的习惯。

## 5. 推荐开发环境

本系列后续默认你有：

- Go 1.22 或更新版本。
- Docker Desktop 或可用的 Docker 环境。
- 一个代码编辑器，例如 GoLand、VS Code、Cursor。
- Git。
- curl 或 Postman。

检查命令：

```bash
go version
docker version
docker compose version
git --version
```

如果某个命令不可用，先把对应工具安装好，再进入后续阶段。

## 6. 推荐项目目录习惯

后续写 demo 时，可以使用这种结构：

```text
rabbitmq-go-learning/
  tutorials/
  demos/
    01-hello-world/
    02-work-queue/
    03-publish-subscribe/
  notes/
  docker-compose.yml
```

Go 项目内部可以这样组织：

```text
cmd/
  producer/
    main.go
  consumer/
    main.go
internal/
  mq/
    connection.go
    publisher.go
    consumer.go
```

初学时不要一开始就过度封装。先把流程跑通，再考虑抽象。

## 7. 本节小结

第 0 阶段你需要补齐的不是 RabbitMQ 细节，而是后端基本功：

- Go 并发和 context。
- 错误处理和优雅退出。
- 数据库事务和幂等。
- Docker 基础。
- 日志、监控、排查意识。

这些能力越扎实，后面学 RabbitMQ 越顺。

