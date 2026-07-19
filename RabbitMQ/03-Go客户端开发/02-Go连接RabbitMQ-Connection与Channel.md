# 02. Go 连接 RabbitMQ：Connection 与 Channel

## 1. 为什么先讲 Connection 和 Channel

RabbitMQ Go 客户端代码里最常见的两个对象是：

```go
*amqp.Connection
*amqp.Channel
```

你必须先理解它们，否则后面所有 producer 和 consumer 都只是机械复制。

## 2. Connection 是什么

Connection 是 Go 程序和 RabbitMQ 之间的 TCP 连接。

```text
Go 程序 -> TCP Connection -> RabbitMQ
```

Connection 成本比较高，通常应该长期复用。

不推荐：

```text
每发一条消息就 Dial 一次
发完立刻 Close
```

推荐：

```text
应用启动时创建 Connection
运行期间复用
应用退出时关闭
```

## 3. Channel 是什么

Channel 是建立在 Connection 上的 AMQP 信道。

可以理解为：

```text
Connection 是一条 TCP 连接
Channel 是这条连接上的逻辑通道
```

Go 客户端通过 Channel 执行：

- 声明 queue。
- 声明 exchange。
- 声明 binding。
- 发布消息。
- 消费消息。
- 设置 QoS。
- ack/nack。

## 4. 最小连接代码

创建：

```text
cmd/connect_check/main.go
```

代码：

```go
package main

import (
	"log"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"

func main() {
	conn, err := amqp.Dial(rabbitURL)
	if err != nil {
		log.Fatalf("dial rabbitmq: %v", err)
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("open channel: %v", err)
	}
	defer ch.Close()

	log.Println("connected to RabbitMQ and opened a channel")
}
```

运行：

```powershell
go run .\cmd\connect_check
```

预期输出：

```text
connected to RabbitMQ and opened a channel
```

## 5. 在 Management UI 中观察连接

运行程序时，打开：

```text
http://localhost:15672
```

进入：

```text
Connections
Channels
```

如果程序很快退出，你可能来不及看到连接。可以临时加：

```go
select {}
```

让程序不退出，再去 UI 观察。

注意：观察完要用 `Ctrl+C` 停止程序。

## 6. 常见连接错误

### 6.1 RabbitMQ 没启动

错误类似：

```text
dial tcp [::1]:5672: connectex: No connection could be made
```

检查：

```powershell
docker ps
```

启动：

```powershell
docker start rabbitmq-dev
```

### 6.2 用户名或密码错误

错误类似：

```text
Exception (403) Reason: "username or password not allowed"
```

检查连接地址：

```text
amqp://go_learner:go_learner_pwd@localhost:5672/
```

### 6.3 vhost 权限错误

如果连接自定义 vhost，但用户没有权限，可能会失败。

查看权限：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_permissions -p rabbitmq_learning
```

设置权限：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl set_permissions -p rabbitmq_learning app_order ".*" ".*" ".*"
```

## 7. Channel 出错后怎么办

在 RabbitMQ 中，某些错误会导致 channel 被关闭。

例如你声明一个已经存在的 queue，但参数不一致：

```text
第一次声明 durable=true
第二次声明 durable=false
```

RabbitMQ 会报错，channel 可能不可继续使用。

学习阶段最常见处理方式：

```text
删掉旧 queue，保持声明参数一致，重新运行程序。
```

生产代码里需要更系统的 channel 重建逻辑，后续阶段再学。

## 8. Connection 和 Channel 的使用建议

学习阶段：

```text
每个 demo 一个 connection，一个 channel。
```

生产阶段：

- connection 长期复用。
- 发布和消费可以使用不同 channel。
- 高吞吐场景可能需要 channel 池或更细的设计。
- channel 不是无限创建，数量也要监控。
- 出错后要能重建 connection/channel。

## 9. 本节小结

你要记住：

- Connection 是 TCP 连接，成本较高。
- Channel 是 AMQP 信道，操作都通过它完成。
- Go 客户端通过 `amqp.Dial` 建立连接。
- 通过 `conn.Channel()` 打开 channel。
- 声明参数不一致可能导致 channel 关闭。

下一节完成第一个 Hello World producer 和 consumer。

