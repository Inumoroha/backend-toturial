# 05. 手动创建 Exchange、Queue、Binding

## 1. 本节目标

本节你要在 Management UI 里手动完成：

```text
创建 exchange
创建 queue
创建 binding
```

完成后得到下面结构：

```text
Producer -> learning.direct -> learning.hello.queue -> Consumer
```

其中：

- `learning.direct` 是 exchange。
- `learning.hello.queue` 是 queue。
- `hello` 是 routing key / binding key。

## 2. 创建 exchange

打开浏览器：

```text
http://localhost:15672
```

登录后进入：

```text
Exchanges -> Add a new exchange
```

填写：

```text
Name: learning.direct
Type: direct
Durability: Durable
Auto delete: No
Internal: No
```

然后点击：

```text
Add exchange
```

### 字段解释

| 字段 | 说明 |
| --- | --- |
| Name | exchange 名称 |
| Type | exchange 类型 |
| Durability | 是否持久化 exchange 定义 |
| Auto delete | 没有绑定后是否自动删除 |
| Internal | 是否只允许 RabbitMQ 内部使用 |

学习阶段先选择：

```text
Durable: Yes
Auto delete: No
Internal: No
```

## 3. 创建 queue

进入：

```text
Queues and Streams -> Add a new queue
```

填写：

```text
Type: Classic
Name: learning.hello.queue
Durability: Durable
```

其他选项保持默认即可。

点击：

```text
Add queue
```

### 字段解释

| 字段 | 说明 |
| --- | --- |
| Type | 队列类型，初学使用 Classic |
| Name | queue 名称 |
| Durability | 是否持久化 queue 定义 |

注意：Durable queue 表示队列定义在 RabbitMQ 重启后还存在，但这不等于消息一定不丢。消息是否持久化，还取决于消息本身的 delivery mode 等设置。

可靠性机制后续阶段再深入。

## 4. 创建 binding

进入刚创建的 exchange：

```text
Exchanges -> learning.direct
```

找到：

```text
Bindings
```

在 `Add binding from this exchange` 中填写：

```text
To queue: learning.hello.queue
Routing key: hello
```

点击：

```text
Bind
```

现在你已经创建了路由关系：

```text
learning.direct -- hello --> learning.hello.queue
```

## 5. 用命令查看 exchange

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_exchanges name type durable auto_delete
```

你应该能看到：

```text
learning.direct direct true false
```

## 6. 用命令查看 queue

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name durable messages_ready messages_unacknowledged consumers
```

你应该能看到：

```text
learning.hello.queue true 0 0 0
```

含义是：

- 队列是 durable。
- 当前没有 ready 消息。
- 当前没有 unacked 消息。
- 当前没有消费者。

## 7. 用命令查看 binding

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_bindings
```

你应该能看到类似：

```text
learning.direct exchange learning.hello.queue queue hello []
```

这表示：

```text
learning.direct 通过 routing key hello 绑定到了 learning.hello.queue
```

## 8. 容易犯的错误

### 错误一：exchange 名称写错

发布消息时 exchange 名称必须和创建时一致。

```text
learning.direct
learning_direct
learning.direct.exchange
```

这些都是不同名字。

### 错误二：routing key 不匹配

direct exchange 中，routing key 通常需要精确匹配 binding key。

如果 binding key 是：

```text
hello
```

你发布时写：

```text
hello.go
```

消息就不会进入这个队列。

### 错误三：创建了 queue 但没有 binding

如果 exchange 没有绑定到 queue，消息可能无法路由到目标队列。

记住：

```text
Exchange 和 Queue 之间要靠 Binding 连接。
```

## 9. 本节小结

你已经手动创建了 RabbitMQ 最基础的三件套：

```text
Exchange: learning.direct
Queue: learning.hello.queue
Binding: hello
```

下一节会手动发布一条消息，并从队列取出来。

