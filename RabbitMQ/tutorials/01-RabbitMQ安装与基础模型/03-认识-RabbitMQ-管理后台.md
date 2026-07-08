# 03. 认识 RabbitMQ 管理后台

## 1. 打开 Management UI

浏览器访问：

```text
http://localhost:15672
```

登录：

```text
Username: go_learner
Password: go_learner_pwd
```

Management UI 是 RabbitMQ 学习阶段最重要的观察工具之一。

## 2. 顶部导航栏

登录后，你通常会看到这些页面：

```text
Overview
Connections
Channels
Exchanges
Queues and Streams
Admin
```

不同版本的 UI 文案可能略有变化，但核心功能类似。

## 3. Overview：总览页

Overview 用来看 RabbitMQ 节点整体状态。

重点关注：

- 消息发布速率。
- 消息投递速率。
- 消息确认速率。
- 队列数量。
- 连接数量。
- channel 数量。
- consumer 数量。
- 内存和磁盘状态。

学习阶段重点看：

```text
Queued messages
Message rates
Global counts
```

你现在可能看到的队列和消息都是 0，这是正常的。

## 4. Connections：连接

Connection 是客户端和 RabbitMQ 之间的 TCP 连接。

后面 Go 程序连接 RabbitMQ 时，会在这里出现一条 connection。

重点理解：

```text
Connection 成本较高，应该长期复用。
```

不好的做法：

```text
每发送一条消息就创建一个 connection
发送完立刻关闭
```

更好的做法：

```text
应用启动时创建 connection
运行期间复用
应用退出时关闭
```

## 5. Channels：信道

Channel 是建立在 Connection 上的轻量通信通道。

可以简单理解为：

```text
Connection 是一条 TCP 大路
Channel 是这条大路上的多条车道
```

在 Go 开发中，你会经常创建 channel 来：

- 声明 exchange。
- 声明 queue。
- 绑定 queue。
- 发布消息。
- 消费消息。

## 6. Exchanges：交换机

Exchange 负责接收生产者发送的消息，并根据规则路由到队列。

常见类型：

- direct
- fanout
- topic
- headers

这一阶段重点掌握前三个。

生产者通常不是直接发给 queue，而是发给 exchange：

```text
Producer -> Exchange -> Queue
```

## 7. Queues and Streams：队列

Queue 是消息真正等待消费的地方。

你需要重点观察：

| 指标 | 含义 |
| --- | --- |
| Ready | 等待被消费的消息数量 |
| Unacked | 已投递给消费者，但还没确认的消息数量 |
| Total | Ready + Unacked |
| Consumers | 当前消费者数量 |

初学阶段先记住：

```text
Ready 很多：消息在排队，还没人处理或处理不过来。
Unacked 很多：消息已经给消费者了，但消费者还没 ack。
```

## 8. Admin：管理

Admin 页面主要用于管理：

- Users
- Virtual Hosts
- Permissions
- Policies

本阶段只需要知道：

- User：谁能登录和连接 RabbitMQ。
- Virtual Host：逻辑隔离空间。
- Permission：某个用户在某个 vhost 里能配置、写入、读取哪些资源。

生产环境不要所有服务都用管理员账号。

## 9. 用 HTTP API 检查 UI 是否可用

Management Plugin 除了 UI，还有 HTTP API。

在 PowerShell 执行：

```powershell
curl.exe -u go_learner:go_learner_pwd http://localhost:15672/api/overview
```

如果返回 JSON，说明管理 API 可用。

如果 Windows PowerShell 里 `curl` 行为奇怪，使用 `curl.exe`，不要只写 `curl`。

## 10. 本节小结

你现在应该知道：

- Overview 看整体状态。
- Connections 看客户端连接。
- Channels 看连接上的信道。
- Exchanges 看消息路由入口。
- Queues and Streams 看消息存储和消费状态。
- Admin 管用户、vhost 和权限。

下一节开始系统学习 AMQP 基础模型。

