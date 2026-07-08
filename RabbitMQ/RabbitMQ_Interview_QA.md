# RabbitMQ 常见面试问题与参考答案

> 使用方式：先通读一遍，再把每个问题尝试用自己的话复述。面试时不要机械背诵，要结合你的 Go 项目、Outbox、retry、DLQ、幂等和监控经验来回答。

## 1. 基础概念

### 1.1 RabbitMQ 是什么？

RabbitMQ 是一个消息代理，也就是 message broker。它负责接收生产者发布的消息，根据 exchange、routing key、binding 等规则把消息路由到队列，再由消费者从队列中消费。

它常用于：

- 异步处理。
- 服务解耦。
- 任务队列。
- 事件通知。
- 削峰填谷。
- 失败重试。

在 Go 后端中，RabbitMQ 常用来连接 API 服务和后台 worker，例如订单创建后异步预占库存、发送通知、增加积分。

### 1.2 RabbitMQ 的核心模型是什么？

核心模型是：

```text
Producer -> Exchange -> Queue -> Consumer
```

含义：

- Producer：生产者，发布消息。
- Exchange：交换机，接收消息并根据规则路由。
- Queue：队列，存储消息，等待消费者消费。
- Binding：绑定 exchange 和 queue 的规则。
- Routing Key：生产者发布消息时携带的路由键。
- Consumer：消费者，从 queue 中消费消息。

生产者通常不直接发送到 queue，而是发送到 exchange。消费者通常不从 exchange 消费，而是从 queue 消费。

### 1.3 Exchange 和 Queue 有什么区别？

Exchange 负责路由，Queue 负责存储。

Exchange 不保存消息，它根据 exchange 类型、routing key 和 binding 规则决定消息进入哪些队列。

Queue 保存消息，消费者从 queue 中获取消息并处理。

如果 exchange 没有把消息路由到任何队列，默认情况下消息可能被丢弃，除非生产者使用 mandatory publish 并监听 return。

### 1.4 Binding 是什么？

Binding 是 exchange 和 queue 之间的路由关系。

例如：

```text
order.events -- order.created --> order.created.inventory.queue
```

表示 `order.created.inventory.queue` 这个队列希望从 `order.events` 交换机接收 routing key 为 `order.created` 的消息。

### 1.5 Routing Key 是什么？

Routing Key 是生产者发布消息时带的路由信息。

例如：

```text
order.created
order.paid
notification.email.send
```

Exchange 会根据自己的类型解释 routing key：

- direct：精确匹配 binding key。
- fanout：通常忽略 routing key，广播到所有绑定队列。
- topic：按通配符模式匹配。

### 1.6 RabbitMQ 中的 Connection 和 Channel 有什么区别？

Connection 是客户端和 RabbitMQ 之间的 TCP 连接，成本较高，应该长期复用。

Channel 是建立在 connection 上的 AMQP 信道，成本较低。声明 exchange、声明 queue、发布消息、消费消息、ack/nack 等操作都通过 channel 完成。

不推荐每发一条消息就创建 connection。推荐应用启动时建立连接，运行期间复用，必要时为发布和消费使用不同 channel。

### 1.7 为什么不要频繁创建 Connection？

因为 connection 是 TCP 连接，创建和销毁成本高，会增加 RabbitMQ 和操作系统负担。

频繁创建 connection 可能导致：

- 连接数暴涨。
- 文件句柄耗尽。
- RabbitMQ 管理 UI 中 connection churn 很高。
- 性能下降。
- 触发网络或资源限制。

正确做法是复用 connection，并合理管理 channel。

## 2. Exchange 类型

### 2.1 RabbitMQ 常见 Exchange 类型有哪些？

常见类型：

- direct：精确匹配 routing key。
- fanout：广播到所有绑定队列。
- topic：基于通配符模式匹配 routing key。
- headers：基于 headers 匹配，较少使用。

Go 后端业务中最常用 direct、fanout、topic。

### 2.2 Direct Exchange 的工作原理是什么？

Direct exchange 要求 routing key 精确匹配 binding key。

例如：

```text
binding key: email.send
routing key: email.send -> 匹配
routing key: sms.send -> 不匹配
```

Direct exchange 不一定是一对一。如果多个队列绑定同一个 binding key，一条消息会进入多个队列。

适合：

- 明确任务类型。
- 邮件、短信任务分发。
- 明确业务事件路由。

### 2.3 Fanout Exchange 的工作原理是什么？

Fanout exchange 会把消息广播到所有绑定到它的队列，通常不关心 routing key。

适合：

- 日志广播。
- 多系统都要接收同一个事件。
- 简单发布订阅。

例如用户注册成功后，邮件服务、积分服务、分析服务都要收到消息。

### 2.4 Topic Exchange 的工作原理是什么？

Topic exchange 使用 routing key 和 binding key 的模式匹配。

通配符：

- `*`：匹配一个单词。
- `#`：匹配零个或多个单词。

例子：

```text
order.* 匹配 order.created、order.paid
order.# 匹配 order.created、order.payment.succeeded
*.created 匹配 order.created、user.created
```

Topic exchange 适合业务事件总线，例如订单事件、用户事件、支付事件。

### 2.5 默认 Exchange 是什么？

RabbitMQ 有一个默认 exchange，名字是空字符串 `""`。

每个 queue 创建时，都会自动绑定到默认 exchange，binding key 是 queue 名称。

所以：

```text
exchange: ""
routing key: queue_name
```

可以把消息投递到同名队列。

很多 Hello World 教程看起来像“直接发给队列”，本质上是通过默认 exchange 实现的。

## 3. Go 客户端

### 3.1 Go 中常用哪个 RabbitMQ 客户端？

RabbitMQ 官方团队维护的 AMQP 0-9-1 Go 客户端是：

```text
github.com/rabbitmq/amqp091-go
```

安装：

```bash
go get github.com/rabbitmq/amqp091-go
```

代码中通常：

```go
import amqp "github.com/rabbitmq/amqp091-go"
```

### 3.2 Go 连接 RabbitMQ 的基本流程是什么？

基本流程：

```text
amqp.Dial -> conn.Channel -> 声明 topology -> Publish 或 Consume -> Close
```

示例：

```go
conn, err := amqp.Dial("amqp://user:pass@localhost:5672/")
ch, err := conn.Channel()
```

生产环境还需要：

- 配置化连接地址。
- 断线重连。
- channel 重建。
- topology 重新声明。
- 结构化日志和指标。

### 3.3 Go 中如何声明队列？

使用：

```go
ch.QueueDeclare(name, durable, autoDelete, exclusive, noWait, args)
```

业务队列通常：

```go
ch.QueueDeclare("order.created.inventory.queue", true, false, false, false, nil)
```

含义：

- durable=true：队列定义持久化。
- autoDelete=false：不自动删除。
- exclusive=false：不是连接独占队列。

### 3.4 Go 中如何声明 Exchange？

使用：

```go
ch.ExchangeDeclare(name, kind, durable, autoDelete, internal, noWait, args)
```

例如：

```go
ch.ExchangeDeclare("order.events", "topic", true, false, false, false, nil)
```

业务 exchange 通常 durable=true，autoDelete=false，internal=false。

### 3.5 Go 中如何绑定队列？

使用：

```go
ch.QueueBind(queueName, routingKey, exchangeName, noWait, args)
```

例如：

```go
ch.QueueBind("order.created.inventory.queue", "order.created", "order.events", false, nil)
```

表示队列订阅 `order.events` 中 routing key 为 `order.created` 的消息。

### 3.6 Go 中如何发布消息？

使用：

```go
ch.PublishWithContext(ctx, exchange, routingKey, mandatory, immediate, amqp.Publishing{
    ContentType:  "application/json",
    DeliveryMode: amqp.Persistent,
    MessageId:    messageID,
    Body:         body,
})
```

生产环境建议：

- 使用 `context.WithTimeout`。
- 设置 `DeliveryMode: amqp.Persistent`。
- 设置 `MessageId` 和 `CorrelationId`。
- 开启 publisher confirm。
- 使用 mandatory publish 处理不可路由消息。

### 3.7 Go 中如何消费消息？

使用：

```go
msgs, err := ch.Consume(queueName, consumerTag, autoAck, exclusive, noLocal, noWait, args)
```

重要业务建议：

```go
autoAck=false
```

处理成功后：

```go
d.Ack(false)
```

处理失败时根据策略：

```go
d.Nack(false, false)
d.Nack(false, true)
d.Reject(false)
```

## 4. 消费确认与失败处理

### 4.1 Auto Ack 和 Manual Ack 有什么区别？

Auto ack：RabbitMQ 把消息投递给消费者后，就认为消息处理成功。

风险是消费者刚收到消息就崩溃，消息也已经被 RabbitMQ 删除，可能丢失。

Manual ack：消费者处理成功后主动调用 ack。RabbitMQ 收到 ack 后才删除消息。

重要业务通常使用 manual ack。

### 4.2 Ack 应该在什么时候调用？

应该在业务处理成功之后调用。

正确顺序：

```text
收到消息 -> 解析 -> 幂等检查 -> 执行业务 -> 提交事务 -> ack
```

不要先 ack 再处理业务，否则业务失败时消息已经删除。

### 4.3 `Ack(false)` 中的 false 是什么？

`false` 表示 `multiple=false`，只确认当前这条消息。

如果 `multiple=true`，表示确认当前 delivery tag 及之前所有未确认消息。

业务代码中通常使用：

```go
d.Ack(false)
```

避免误确认多条消息。

### 4.4 Nack 和 Reject 有什么区别？

`Nack` 是负确认，支持 `multiple`。

`Reject` 是拒绝消息，只能拒绝单条。

常见：

```go
d.Nack(false, false)
d.Nack(false, true)
d.Reject(false)
```

业务中更常用 `Nack(false, false)` 配合 DLQ，或发布到 retry 后 ack 原消息。

### 4.5 `Nack(false, true)` 有什么风险？

`requeue=true` 会把消息重新放回队列。

如果消息是毒消息，永远处理失败，就会出现：

```text
失败 -> requeue -> 再消费 -> 再失败 -> 再 requeue
```

这会造成无限循环，打爆消费者。

更好的方案是有限重试，超过次数进入 DLQ。

### 4.6 什么是 Prefetch？

Prefetch 限制消费者可以同时持有多少条未 ack 消息。

Go 中：

```go
ch.Qos(1, 0, false)
```

`prefetch=1` 表示消费者处理完并 ack 一条后，RabbitMQ 再投递下一条。

它常用于 Work Queue，避免慢消费者一次拿太多消息。

### 4.7 Prefetch 设置越大越好吗？

不是。

prefetch 太小：

- 公平性好。
- 吞吐可能受限。

prefetch 太大：

- 吞吐可能更高。
- unacked 可能变高。
- 单个消费者可能压住太多消息。
- 消费者崩溃时重投压力大。

需要根据处理耗时、消费者并发和下游能力压测。

## 5. 生产端可靠性

### 5.1 `PublishWithContext` 返回 nil 是否代表消息可靠到达队列？

不代表。

它通常只能说明客户端发布动作没有立即报错。

它不一定代表：

- RabbitMQ 已经确认接收。
- 消息已经持久化。
- 消息已经路由到业务队列。
- 消费者已经处理。

生产端可靠性需要：

- publisher confirm。
- mandatory publish。
- durable exchange。
- durable queue。
- persistent message。

### 5.2 Publisher Confirm 解决什么问题？

Publisher confirm 让 RabbitMQ 告诉生产者：

```text
这条发布的消息 broker 已经确认。
```

Go 中：

```go
ch.Confirm(false)
confirms := ch.NotifyPublish(...)
```

Confirm ack 只说明 broker 确认消息，不说明消费者处理成功。

### 5.3 Confirm 超时怎么办？

Confirm 超时时，消息状态未知：

- 可能 RabbitMQ 已收到，但 confirm 响应丢了。
- 也可能 RabbitMQ 没收到。

通常生产者会重试。

这可能造成重复消息，所以消费者必须幂等。

### 5.4 Mandatory Publish 解决什么问题？

Mandatory publish 用来发现不可路由消息。

如果：

```text
mandatory=true
```

且消息没有匹配到任何队列，RabbitMQ 会把消息 return 给生产者。

Go 中监听：

```go
returns := ch.NotifyReturn(...)
```

它解决的是：

```text
消息发布到 exchange，但没有路由到任何 queue。
```

### 5.5 Publisher Confirm 和 Mandatory Publish 有什么区别？

Publisher confirm 关注 broker 是否确认发布。

Mandatory return 关注消息是否不可路由。

二者解决的问题不同，重要业务通常都要考虑。

### 5.6 durable queue 是否代表消息一定不丢？

不代表。

durable queue 只表示队列定义在 RabbitMQ 重启后仍然存在。

消息要更可靠，还需要：

- durable exchange。
- durable queue。
- persistent message。
- publisher confirm。
- manual ack。

### 5.7 persistent message 是否代表消息绝对不丢？

不代表绝对不丢。

Persistent message 表示消息需要持久化，但生产者仍然需要 publisher confirm 确认 broker 接收。

可靠性是链路，不是单个配置。

## 6. 重试和死信

### 6.1 什么是 DLX 和 DLQ？

DLX 是 Dead Letter Exchange，死信交换机。

DLQ 是 Dead Letter Queue，死信队列。

消息在以下情况可能变成死信：

- 被 reject/nack 且 requeue=false。
- 消息 TTL 过期。
- 队列超过长度限制。
- quorum queue 达到投递限制等。

死信会被重新发布到配置的 DLX，再路由到 DLQ。

### 6.2 为什么需要 DLQ？

DLQ 用于保存自动处理失败的消息，方便排查和人工补偿。

没有 DLQ 时，失败消息可能被丢弃，排查困难。

DLQ 不是垃圾桶。DLQ 有消息应该告警。

### 6.3 如何设计延迟重试？

常见方案：

```text
业务队列 -> 消费失败 -> retry queue
retry queue 设置 TTL
TTL 到期 -> dead-letter 回业务 exchange
再次进入业务队列
超过最大次数 -> DLQ
```

重点：

- 不要无限 requeue。
- retry count 要记录。
- 超过最大次数进入 DLQ。
- 发布到 retry 成功后再 ack 原消息。

### 6.4 为什么不直接用 `Nack(false, true)` 做重试？

因为它会立即重新入队，可能形成毒消息循环。

如果错误需要等待恢复，例如数据库故障、外部 API 超时，应该延迟重试，而不是立刻 requeue。

### 6.5 retry count 放在哪里？

常见放在 message headers：

```text
x-retry-count
```

也可以放在消息体或数据库记录中。

如果使用 Outbox 或业务表，也可以记录在数据库里，方便查询和治理。

## 7. 幂等和一致性

### 7.1 为什么消费者必须幂等？

RabbitMQ 可靠消费通常是至少一次投递。

消息可能重复：

- 消费者处理成功但 ack 前崩溃。
- ack 包丢失。
- 生产者 confirm 超时后重试。
- retry queue 重新投递。
- Outbox Worker 重复发布。

所以消费者必须保证重复消息不会造成重复业务副作用。

### 7.2 如何设计幂等键？

优先使用业务唯一键，而不是只用 message_id。

例如：

```text
order:{order_id}:points:add
order:{order_id}:inventory:reserve
notification:{notification_id}:send
```

因为同一个业务事件可能被重新包装成不同 message_id，但业务上仍然只能处理一次。

### 7.3 幂等常见实现方式有哪些？

常见：

- 数据库唯一索引。
- processed_messages 表。
- 业务状态机。
- Redis SET NX 短期去重。
- 业务流水唯一约束。

关键业务推荐数据库唯一索引，幂等记录和业务变更放在同一个事务里。

### 7.4 什么是 Transactional Outbox？

Transactional Outbox 是事务性发件箱模式。

核心思想：

```text
业务表和 outbox_messages 表在同一个数据库事务中写入。
```

然后由 Outbox Worker 扫描 outbox 表，发布 RabbitMQ。

它解决：

```text
数据库事务成功但消息发布失败。
```

### 7.5 Outbox 是否能保证消费者只处理一次？

不能。

Outbox 解决的是业务数据和“待发布消息”的一致性。

如果 Outbox Worker 发布成功但标记 sent 前崩溃，消息可能再次发布。

所以消费者仍然必须幂等。

### 7.6 Outbox Worker 如何设计？

基本流程：

```text
扫描 pending outbox
标记 publishing
发布 RabbitMQ
等待 publisher confirm
成功 -> 标记 sent
失败 -> retry_count + 1，设置 next_retry_at
超过次数 -> dead
```

需要监控：

- pending 数量。
- 最老 pending 时间。
- dead 数量。
- 发布失败次数。

## 8. 业务场景

### 8.1 如何设计订单事件系统？

推荐：

```text
Exchange: order.events
Type: topic

Routing keys:
order.created
order.paid
order.cancelled

Queues:
order.created.inventory.queue
order.paid.points.queue
order.notify.queue
order.all.audit.queue
```

生产者发布业务事实，消费者自己订阅。

订单服务不应该直接发布 `call_inventory_service` 这种命令式消息。

### 8.2 如何设计通知系统？

通知通常是 task。

推荐：

```text
Exchange: notification.task.direct
Type: direct

Routing keys:
notification.email.send
notification.sms.send
notification.inbox.send

Queues:
notification.email.worker.queue
notification.sms.worker.queue
notification.inbox.worker.queue
```

使用 `notification_id` 做幂等，失败进入 retry，超过次数进入 DLQ。

### 8.3 如何实现订单超时取消？

常见方案是 TTL + DLX。

流程：

```text
创建订单
发送 order.timeout.check 到 delay queue
delay queue TTL 到期
dead-letter 到 order.task.exchange
进入 order.timeout.check.queue
timeout-worker 查询订单状态
如果仍 pending_payment，则条件取消
```

取消时必须使用条件更新：

```sql
UPDATE orders
SET status = 'cancelled'
WHERE id = ? AND status = 'pending_payment';
```

防止取消已支付订单。

## 9. 消息堆积和线上排查

### 9.1 消息堆积怎么看？

看队列的：

- messages_ready。
- messages_unacknowledged。
- consumers。
- publish rate。
- deliver rate。
- ack rate。

如果 publish rate 长期大于 ack rate，队列会堆积。

### 9.2 ready 很高说明什么？

ready 高表示消息还在队列中等待投递或消费。

常见原因：

- 没有消费者。
- 消费者数量不足。
- 消费者处理慢。
- 下游服务慢。
- 生产速度过高。

处理：

- 启动或扩容消费者。
- 检查消费者日志。
- 检查下游数据库/API。
- 暂停非核心生产者。
- 处理毒消息。

### 9.3 unacked 很高说明什么？

unacked 高表示消息已经投递给消费者，但还没 ack。

常见原因：

- 消费者处理慢。
- 消费者卡住。
- 忘记 ack。
- prefetch 太大。
- 下游服务阻塞。

处理：

- 查看消费者日志。
- 检查处理耗时。
- 调整 prefetch。
- 重启异常消费者。
- 优化下游依赖。

### 9.4 consumers 为 0 说明什么？

说明当前队列没有消费者。

可能原因：

- 消费者进程挂了。
- 部署失败。
- 连接 RabbitMQ 失败。
- vhost 或权限错误。
- queue 名称配置错误。

关键业务队列 consumers 为 0 应该告警。

### 9.5 连接数暴涨怎么办？

先查来源：

```text
rabbitmqctl list_connections name user peer_host state channels
```

可能原因：

- 应用每条消息创建连接。
- 重连风暴。
- 连接泄漏。
- 某个服务异常扩容。

处理：

- 找到来源服务。
- 回滚异常版本。
- 修复连接复用。
- 限制连接来源。

### 9.6 DLQ 有消息怎么办？

不要直接清空。

处理步骤：

1. 抽样查看消息。
2. 根据 message_id 查日志。
3. 查看 retry count 和 last error。
4. 判断是代码 bug、数据问题还是外部依赖故障。
5. 修复后决定重放、补偿或归档。

DLQ 有消息应该告警。

## 10. 监控和运维

### 10.1 RabbitMQ 需要监控哪些指标？

核心：

- messages_ready。
- messages_unacknowledged。
- consumers。
- publish rate。
- deliver rate。
- ack rate。
- connection count。
- channel count。
- memory。
- disk。
- file descriptors。
- DLQ 消息数。

业务侧还要监控：

- Outbox pending。
- Outbox oldest pending age。
- 消费成功率。
- 消费失败率。
- 消费耗时。
- retry 次数。

### 10.2 Management UI 能替代 Prometheus 吗？

不能。

Management UI 适合临时查看，不适合长期指标采集、历史趋势和告警。

生产环境推荐 RabbitMQ Prometheus Plugin + Prometheus + Grafana + Alertmanager。

### 10.3 RabbitMQ 权限如何设计？

使用：

- vhost 隔离环境或业务。
- 不同服务使用不同 user。
- configure/write/read 按需授权。
- 不使用管理员账号跑业务服务。

生产者通常只需要 write。

消费者通常需要 read，必要时需要 write retry/DLQ exchange。

### 10.4 生产环境为什么不能用 guest/guest？

`guest/guest` 只适合本地学习。

生产环境应该：

- 删除或禁用默认账号。
- 使用强密码。
- 使用最小权限账号。
- 限制 Management UI 访问。
- 密码通过 Secret 或配置系统管理。

## 11. 高可用和队列类型

### 11.1 Classic Queue 和 Quorum Queue 有什么区别？

Classic Queue 是传统队列，适合普通任务和一般业务消息。

Quorum Queue 是基于 Raft 的 replicated durable queue，更适合关键业务消息和高可用场景。

Quorum Queue 数据安全更强，但资源消耗更高，写入成本更大，不适合大量临时队列。

### 11.2 什么场景适合 Quorum Queue？

适合：

- 支付成功事件。
- 订单关键状态事件。
- 资金流水。
- 账务消息。
- 不能接受单节点队列故障风险的消息。

不一定适合：

- 普通邮件通知。
- 临时订阅队列。
- 大量短生命周期队列。

### 11.3 RabbitMQ 集群是否等于消息自动复制？

不等于。

集群共享元数据，例如 exchange、queue、binding、user、vhost 等。

消息数据是否复制取决于队列类型和配置。

关键业务要考虑 Quorum Queue。

### 11.4 RabbitMQ Streams 适合什么？

RabbitMQ Streams 适合：

- 高吞吐事件流。
- 消息保留。
- 可重复读取。
- 多消费者从不同位置消费。

普通 queue 更适合一次性任务处理和业务消息分发。

如果核心需求是长期保留和大规模回放，也可以考虑 Kafka。

## 12. 架构取舍

### 12.1 RabbitMQ 和 Kafka 怎么选？

RabbitMQ 更适合：

- 业务消息。
- 任务队列。
- 复杂路由。
- 低延迟异步处理。
- retry/DLQ。

Kafka 更适合：

- 高吞吐日志流。
- 用户行为埋点。
- 数据管道。
- 长期保留和回放。
- 多消费组按 offset 独立读取。

一句话：

```text
RabbitMQ 是业务消息代理，Kafka 是分布式事件日志。
```

### 12.2 RabbitMQ 和 Redis Streams 怎么选？

RabbitMQ 更适合完整消息系统：

- exchange 路由。
- 队列治理。
- retry/DLQ。
- 权限和管理 UI。

Redis Streams 更适合：

- 已有 Redis。
- 轻量消息流。
- 简单 consumer group。
- 小规模任务。

如果消息系统是核心基础设施，RabbitMQ 更专业。

### 12.3 RabbitMQ 和数据库任务表怎么选？

数据库任务表适合：

- 任务量小。
- 强依赖数据库事务。
- 路由简单。
- 可接受秒级延迟。
- 不想增加中间件。

RabbitMQ 适合：

- 多服务订阅。
- 复杂路由。
- 更高实时性。
- 独立扩展消费者。
- retry/DLQ 治理。

数据库任务表不是低级方案，它只是适合更简单的场景。

### 12.4 RabbitMQ 是否适合做 RPC？

RabbitMQ 可以实现 RPC，例如使用 `reply_to` 和 `correlation_id`。

但不建议默认用 RabbitMQ 做 RPC。

如果调用方必须立即拿到结果，HTTP/gRPC 通常更简单。

RabbitMQ RPC 适合少数必须通过消息系统承载请求响应的场景。

## 13. 面试项目表达

### 13.1 如何介绍你的 RabbitMQ 项目？

可以这样说：

```text
我做过一个 Go + RabbitMQ 的异步订单系统。订单 API 不直接发布消息，而是在同一个数据库事务里写 orders 和 outbox_messages。Outbox Worker 通过 publisher confirm 可靠发布 order.created、order.paid 等事件。消费者使用 manual ack、prefetch、retry queue 和 DLQ 处理失败。积分、库存、通知消费者通过业务唯一键和数据库唯一索引做幂等。订单超时取消用 TTL + DLX 实现，并通过状态机条件更新避免取消已支付订单。
```

### 13.2 如何回答“你如何保证消息不丢”？

从链路回答：

```text
生产端使用 publisher confirm 确认 broker 接收，mandatory publish 处理不可路由消息。RabbitMQ 侧使用 durable exchange、durable queue 和 persistent message。消费端使用 manual ack，业务成功后再 ack。数据库和消息一致性用 Transactional Outbox。失败消息进入 retry 和 DLQ。
```

同时补充：

```text
可靠投递通常意味着至少一次，所以消费者必须幂等。
```

### 13.3 如何回答“消息重复怎么办”？

```text
重复消息是至少一次投递的正常风险。我会用业务唯一键做幂等，比如 order_id + event_type + action，并用数据库唯一索引保证同一业务动作只执行一次。重复消息识别后直接 ack，不进入 DLQ。
```

### 13.4 如何回答“消息堆积怎么办”？

```text
先看 ready、unacked、consumers、publish rate、ack rate。ready 高通常是消费跟不上或没有消费者；unacked 高通常是消费者处理慢、卡住或忘记 ack；consumers 为 0 说明消费者掉线。处理上可以扩容消费者、检查下游依赖、降低生产速率、处理毒消息和 DLQ。
```

## 14. 高频简答速记

### Exchange 做什么？

路由消息。

### Queue 做什么？

存储消息，等待消费。

### Binding 做什么？

连接 exchange 和 queue，定义路由规则。

### Direct 是什么？

routing key 精确匹配 binding key。

### Fanout 是什么？

广播到所有绑定队列。

### Topic 是什么？

使用 `*` 和 `#` 做模式匹配。

### Ack 是什么？

消费者确认消息处理成功。

### Nack 是什么？

消费者告诉 RabbitMQ 消息处理失败。

### Prefetch 是什么？

限制消费者同时持有的未 ack 消息数量。

### Publisher Confirm 是什么？

RabbitMQ 对生产者发布消息的确认。

### Mandatory Publish 是什么？

让生产者感知不可路由消息。

### DLQ 是什么？

保存最终失败消息的死信队列。

### Outbox 是什么？

业务表和待发布消息表同事务写入，解决数据库事务和消息发布一致性。

### 幂等是什么？

同一业务操作执行一次和多次结果一致。

## 15. 中高级追问与细节辨析

### 15.1 RabbitMQ 能保证 exactly once 吗？

不能简单地说能。

RabbitMQ 常见可靠性组合是至少一次投递：publisher confirm、durable queue、persistent message、manual ack、retry/DLQ。这个组合能显著降低消息丢失概率，但在网络超时、确认丢失、消费者处理成功但 ack 失败、Outbox Worker 重试发布等场景下，都可能出现重复消息。

所以更准确的回答是：

```text
RabbitMQ 侧通常提供至少一次语义。业务要通过幂等键、唯一索引、状态机条件更新、去重表等手段，把最终效果做成接近 exactly once 的业务结果。
```

### 15.2 at most once、at least once、effectively once 有什么区别？

- at most once：最多处理一次，可能丢消息。例如 auto ack 下消费者拿到消息后宕机，RabbitMQ 已经认为消息成功。
- at least once：至少处理一次，尽量不丢，但可能重复。例如 manual ack + publisher confirm。
- effectively once：底层仍可能重复，但业务通过幂等和事务边界，让最终业务结果只生效一次。

面试中不要承诺 RabbitMQ 天然 exactly once。更稳的说法是：消息链路按至少一次设计，业务结果按幂等设计。

### 15.3 为什么不能在数据库事务里直接 publish 消息？

因为数据库事务和 RabbitMQ 发布不是同一个事务资源。

典型风险：

- 数据库提交成功，但 publish 失败，业务数据有了，消息没发出去。
- publish 成功，但数据库提交失败，消费者收到了不存在或不该发生的业务事件。
- publish 超时，生产者不知道 broker 是否已经收到消息，重试可能产生重复。

更推荐：

```text
在同一个数据库事务里写业务表和 outbox_messages 表。事务提交后，由 Outbox Worker 扫描待发送消息，发布到 RabbitMQ，并用 publisher confirm 更新发送状态。
```

### 15.4 Outbox Worker 发布成功但更新 outbox 状态失败怎么办？

这会导致 Worker 下次再次扫描到同一条 outbox 记录并重复发布。

因此消费者必须幂等。Outbox 解决的是“业务数据提交”和“消息最终可发布”的一致性问题，不解决“消费者只处理一次”的问题。

推荐回答：

```text
Outbox Worker 发布成功但更新状态失败时，宁可重复发布，也不能丢消息。重复由消费者幂等处理。outbox 表可以记录 published_at、retry_count、last_error，并配合唯一 message_id 或 event_id。
```

### 15.5 RabbitMQ 如何保证消息顺序？

RabbitMQ 只能在非常受限的条件下保持顺序：

- 单个队列。
- 单个消费者。
- prefetch 接近 1。
- 消费者串行处理。
- 不发生 nack requeue、重试、消费者重启。

一旦有多个消费者、并发处理、失败重试、消息重新入队，业务顺序就可能被打乱。

所以订单类场景常用：

- 按 `order_id` 分片路由到固定队列。
- 同一订单内用状态机防止倒退。
- 事件携带版本号或状态更新时间。
- 消费者执行条件更新，例如 `where status = 'created'`。

### 15.6 单队列多消费者为什么会破坏业务顺序？

RabbitMQ 从队列按顺序投递消息，但多个消费者处理速度不同。

例如消息 A 先投递给消费者 1，消息 B 后投递给消费者 2。如果消费者 2 处理更快，B 的业务效果可能先于 A 生效。

所以多消费者提升吞吐，不等价于保持业务顺序。

### 15.7 prefetch 对顺序有什么影响？

prefetch 越大，单个消费者一次拿到的未 ack 消息越多，吞吐可能更高，但失败重试、消费者宕机时的重新投递窗口也更大。

如果面试官问“要严格顺序怎么办”，回答应该偏保守：

```text
单队列、单消费者、prefetch=1、串行处理，并在业务侧加状态机。若吞吐不足，再按业务 key 分片成多队列，每个 key 内部保持顺序。
```

### 15.8 `Nack(false, true)` 为什么容易造成重试风暴？

因为它会把失败消息重新放回队列。如果失败原因是稳定存在的，例如参数错误、下游服务不可用、数据库约束冲突，消息可能被立即再次消费、再次失败、再次 requeue。

结果是：

- CPU 空转。
- 日志刷屏。
- 正常消息被阻塞。
- 消息不断 redelivered。
- 下游依赖被反复打爆。

更好的方式是进入有间隔的 retry queue，超过次数后进入 DLQ。

### 15.9 什么是毒消息？

毒消息是指无论重试多少次都无法成功处理的消息。

常见原因：

- 消息格式错误。
- 必填字段缺失。
- 业务状态已经不允许处理。
- 依赖的数据不存在。
- 消费者代码 bug。

处理策略：

- 记录失败原因。
- 限制最大重试次数。
- 进入 DLQ。
- 提供人工修复和重放工具。
- 修复后重新投递，不能无限 requeue。

### 15.10 DLQ 消息如何重放？

不要直接把 DLQ 全量倒回原队列。正确流程是：

1. 先看 DLQ 的消息数量、失败原因、时间范围。
2. 抽样检查消息体、headers、routing key、`x-death` 信息。
3. 判断是代码 bug、配置错误、下游故障还是脏数据。
4. 修复根因。
5. 小批量重放。
6. 观察成功率、重试率、堆积、下游压力。
7. 再逐步放大。

面试回答重点：DLQ 是隔离和诊断机制，不是垃圾桶。

### 15.11 `x-death` 有什么用？

`x-death` 是 RabbitMQ 在消息死信转发时追加的 header 信息，通常可以看到消息因为什么原因进入死信、经过哪些队列、死信次数等。

它适合用来：

- 判断重试次数。
- 排查失败路径。
- 区分 expired、rejected、maxlen 等死信原因。
- 做 DLQ 分析和报警。

注意：业务不要完全依赖某个 header 格式做强耦合逻辑，关键重试次数也可以放在业务 header 中自己维护。

### 15.12 队列声明参数不一致会怎样？

同名 queue 或 exchange 如果已经存在，再用不兼容的参数声明，RabbitMQ 通常会关闭 channel 并返回类似 `PRECONDITION_FAILED` 的错误。

例如同一个队列第一次声明为 durable，第二次用 non-durable 声明，就会出问题。

所以拓扑参数要统一管理，生产者和消费者不能各写一套互相冲突的声明逻辑。

### 15.13 为什么拓扑声明要做成幂等？

服务启动时声明 exchange、queue、binding 是常见做法。幂等声明可以保证服务多次启动、多个实例同时启动时，不会因为资源已存在而失败。

但是“幂等”要求声明参数一致。名字相同但参数不同，不是幂等，而是配置冲突。

### 15.14 Channel 被关闭后还能继续使用吗？

不能。

`amqp091-go` 文档中也强调，channel 上的方法一旦返回错误，这个 channel 就应该被视为无效，需要重新创建 channel，并重新声明必要拓扑。

生产代码中要监听：

- `Connection.NotifyClose`
- `Channel.NotifyClose`
- `Channel.NotifyReturn`
- `Channel.NotifyPublish`

### 15.15 为什么必须持续接收 NotifyPublish、NotifyReturn 这类异步事件？

因为这些通知通道如果没有接收者，客户端库的同步方法可能被阻塞。

生产级 publisher 开启 confirm 后，应该有专门 goroutine 持续读取 confirm 事件，维护 delivery tag 和业务 message id 的映射，并处理 ack、nack、超时。

### 15.16 `amqp091-go` 的 Channel 能被多个 goroutine 共享吗？

不建议，也不应该把同一个 `*amqp.Channel` 给多个 goroutine 并发调用。

官方 Go package 文档明确提示：Channel 不是线程安全的。生产代码更推荐：

- 一个 publisher worker 独占一个 channel。
- 一个 consumer 或一组串行消费逻辑独占一个 channel。
- 发布和消费使用不同 channel。
- channel 出错后丢弃并重建。

### 15.17 为什么发布和消费不要混用同一个 Channel？

发布、消费、ack、confirm、return 都会在 channel 上发生。如果混在一起，会让流量、错误和异步事件处理变复杂。

更清晰的设计：

- publisher channel：只负责 publish、confirm、return。
- consumer channel：只负责 consume、ack、nack、qos。
- topology channel：启动时声明拓扑，用完可关闭。

这样出错隔离更好，监控和重连逻辑也更清楚。

### 15.18 Connection 和 Channel 应该如何复用？

Connection 是 TCP 连接，成本较高，应用级别长期复用。

Channel 是 AMQP 信道，成本较低，可以按用途创建多个。

典型结构：

```text
一个进程维护少量 connection
每类 publisher/consumer 创建独立 channel
channel 出错重建
connection 出错时整体重连，并重建所有 channel 和拓扑
```

### 15.19 RabbitMQ 的 flow control 是什么？

当 RabbitMQ 节点内存、磁盘或内部资源压力过大时，可能会对发布端施加流控，减缓或阻塞生产者继续发布。

应用侧应该：

- 监控 connection blocked 状态。
- 记录 publish 延迟。
- 设置发布超时。
- 做背压和限流。
- 不要无脑无限重试。

### 15.20 memory alarm 和 disk alarm 出现时会怎样？

RabbitMQ 会保护自身资源，可能阻塞发布、降低吞吐，甚至导致集群状态异常。

排查方向：

- 队列是否堆积。
- 是否有大量 unacked。
- 消费者是否掉线或处理变慢。
- 是否有大消息。
- 磁盘是否不足。
- DLQ 或 retry 队列是否无限增长。

短期处理是止血：限流生产者、恢复消费者、扩容资源、转移或清理积压。长期处理是容量规划和告警规则。

### 15.21 RabbitMQ 适合传大消息吗？

不适合把 RabbitMQ 当大文件传输系统。

大消息会带来：

- 网络传输慢。
- 内存压力大。
- 磁盘占用高。
- confirm 延迟上升。
- 消费者处理时间长。

推荐做法：

```text
大文件放对象存储，消息里只放文件 URL、对象 key、校验信息和业务元数据。
```

### 15.22 Priority Queue 适合解决所有优先级问题吗？

不适合。

优先级队列会增加 broker 的内部管理成本，也可能影响吞吐。它适合少量明确等级的任务优先级，不适合用很多优先级模拟复杂调度系统。

很多业务可以通过拆分队列实现：

- high.priority.queue
- normal.priority.queue
- low.priority.queue

然后消费者按资源比例消费不同队列。

### 15.23 Queue Length Limit 有什么用？

队列长度限制用于防止队列无限增长拖垮 broker。

常见策略：

- 超过长度后丢弃旧消息。
- 拒绝新消息。
- 配合 DLX 把溢出的消息转移到死信。

面试回答要强调：长度限制是保护措施，不是常规流量治理的替代品。真正的问题仍然是生产速率、消费能力和下游依赖。

### 15.24 AMQP 事务和 Publisher Confirm 怎么选？

实际项目优先使用 publisher confirm。

AMQP 事务会显著降低吞吐，因为每次提交都需要事务确认。Publisher confirm 更适合高吞吐可靠发布，可以单条确认，也可以批量确认。

常见回答：

```text
业务一致性用 Outbox，消息发布可靠性用 publisher confirm，而不是依赖 AMQP 事务。
```

### 15.25 Confirm ack 慢说明什么？

可能原因：

- broker 磁盘写入慢。
- quorum queue 复制延迟。
- 队列堆积严重。
- 网络延迟。
- 消息太大。
- broker 正在流控。

排查时要同时看 publish rate、confirm latency、disk I/O、queue depth、node resource、网络和队列类型。

### 15.26 mandatory return 和 confirm 的顺序能完全依赖吗？

不要在业务逻辑上依赖不同异步通知通道之间的严格顺序。

`mandatory` 处理不可路由，`confirm` 处理 broker 是否接收发布。生产代码应同时监听 return 和 confirm，并用 message id 或 publish sequence 做关联。

更稳的设计是：不可路由直接视为发布失败，记录错误并触发告警。

### 15.27 RabbitMQ 集群是否自动解决所有高可用问题？

不是。

集群让多个节点组成一个 broker 集群，但队列数据是否复制取决于队列类型和配置。

高可用还要考虑：

- 使用 quorum queue 或 streams 这类复制数据结构。
- 客户端连接多个节点或通过负载均衡。
- 节点故障演练。
- 网络分区处理。
- 备份和恢复。
- 监控告警。

### 15.28 Quorum Queue 为什么比 Classic Queue 更适合关键业务队列？

Quorum Queue 基于 Raft 复制，目标是数据安全和高可用，适合订单、支付、库存这类关键消息。

但它也有成本：

- 写入需要复制到多数派。
- 延迟通常比非复制队列更高。
- 不适合高 churn 临时队列。
- 不适合超大积压场景。

面试回答要体现取舍：关键业务用 quorum，一般临时任务或非关键场景可以继续用 classic，审计和回放考虑 streams。

### 15.29 Classic Queue Mirroring 还能作为高可用方案吗？

不要把 classic mirrored queues 当作新项目高可用方案。

RabbitMQ 4.0 起移除了 classic queue mirroring。新项目需要复制和高可用时，应优先评估 quorum queues 或 streams。

### 15.30 RabbitMQ Streams 和普通 Queue 的核心区别是什么？

普通 queue 更像“消息被消费后删除”的任务队列，目标通常是趋向清空。

Streams 更像可重复读取的追加日志，消息按保留策略保存，消费者可以从指定 offset 读取，适合：

- 事件审计。
- 消息回放。
- 大 backlog。
- 多消费者重复读取。
- 高吞吐日志式场景。

不适合把 streams 简单理解成更快的 queue，它的语义不同。

### 15.31 Federation 和 Shovel 有什么区别？

Shovel 更像消息搬运工，把一个 broker 或 queue 的消息搬到另一个地方，适合迁移、桥接、临时同步。

Federation 更像跨 broker 的联邦关系，适合 exchange 或 queue 之间的跨集群消息流动。

两者都会引入重复、延迟、顺序和故障恢复问题，跨地域设计时必须接受最终一致。

### 15.32 RabbitMQ 做 RPC 有什么问题？

RabbitMQ 支持请求响应模式，但不要滥用。

风险：

- 调用链变长。
- 超时和重试复杂。
- 响应队列管理复杂。
- 失败定位困难。
- 容易把异步系统写成更难排查的同步调用。

适合低频、明确边界的内部请求响应。不适合替代所有 HTTP/gRPC 调用。

### 15.33 消费者宕机时 unacked 消息会怎样？

如果使用 manual ack，消费者所在 channel 或 connection 关闭后，未 ack 的消息会被 RabbitMQ 重新入队，之后投递给其他消费者。

这也是为什么消费者必须幂等：消费者可能已经执行完业务逻辑，但还没来得及 ack 就宕机，消息会被重新投递。

### 15.34 Delivery Tag 的作用域是什么？

Delivery tag 用来标识 channel 上的一次投递。它的作用域是 channel，不是全局。

所以不能拿一个 channel 上的 delivery tag 到另一个 channel 上 ack，否则会出现 unknown delivery tag 之类错误，严重时 channel 会被关闭。

### 15.35 multi ack 有什么风险？

`Ack(tag, true)` 会确认当前 tag 以及之前所有未确认消息。

如果你的消费者并发处理消息，后面的消息先处理完、前面的消息还没处理完，随便 multi ack 可能把尚未成功处理的消息一起确认掉。

所以并发消费者中通常使用 `Ack(false)` 单条确认，除非你能严格保证处理顺序。

### 15.36 消费者处理慢应该先调大 prefetch 吗？

不一定。

先判断慢在哪里：

- CPU 计算慢：扩容消费者或优化代码。
- 数据库慢：优化 SQL、索引、连接池。
- 下游接口慢：限流、超时、熔断。
- 单条消息太大：拆分消息。
- 消费者串行瓶颈：增加并发。

prefetch 只是控制“预取未 ack 消息数量”，不是万能加速器。调大 prefetch 会增加消费者内存和重复投递窗口。

### 15.37 如何设计消费者优雅退出？

推荐流程：

1. 收到 SIGTERM。
2. 停止接收新消息或取消 consumer。
3. 等待正在处理的消息完成。
4. 成功的消息 ack。
5. 未完成的消息不 ack，让连接关闭后自动重新入队。
6. 关闭 channel 和 connection。

在 Kubernetes 中还要配合 `terminationGracePeriodSeconds` 和 readiness probe。

### 15.38 如何回答“RabbitMQ 消息丢了你怎么排查”？

按链路拆：

- 生产者是否真的调用 publish。
- publish 是否返回错误。
- 是否开启 publisher confirm。
- confirm 是否 ack。
- mandatory return 是否有不可路由消息。
- exchange、binding、routing key 是否正确。
- queue 是否 durable。
- message 是否 persistent。
- 消费者是否 auto ack。
- 消费者是否业务失败后错误 ack。
- 是否 TTL 过期、队列溢出或进入 DLQ。
- broker 是否重启、磁盘告警、节点故障。

最后再看日志、监控、DLQ、审计表、Outbox 状态。

### 15.39 如何回答“RabbitMQ 重复消费你怎么排查”？

常见原因：

- 消费者处理成功但 ack 失败。
- 消费者超时或宕机导致未 ack 消息重新投递。
- 生产者超时后重复发布。
- Outbox Worker 重复扫描。
- retry/DLQ 重放。
- 网络抖动。

解决方式不是追求“永不重复”，而是明确幂等键、去重表、唯一索引和状态机。

### 15.40 面试回答 RabbitMQ 问题的通用框架是什么？

建议按这个顺序回答：

```text
先说核心概念
再说可靠性机制
再说业务落地
再说风险和取舍
最后说监控排查
```

例如问“怎么保证消息可靠”，不要只说 durable。要从生产端、broker、消费端、业务一致性、幂等、监控完整回答。

## 16. RabbitMQ 4.x 版本意识

### 16.1 为什么面试中要有版本意识？

因为 RabbitMQ 的一些能力和推荐实践会随版本变化。

你不需要背所有版本差异，但要能说：

```text
具体参数和能力要以公司 RabbitMQ 版本和官方文档为准。我会先确认线上版本，再决定是否使用 quorum queue、streams、consumer timeout、delayed retry 等能力。
```

### 16.2 RabbitMQ 4.x 中高可用队列应该重点关注什么？

重点关注 quorum queues 和 streams。

RabbitMQ 4.0 起 classic queue mirroring 已被移除。需要复制和高可用的关键队列，应优先评估 quorum queue；需要保留、回放、大 backlog 或日志式消费的场景，应评估 streams。

### 16.3 Quorum Queue 在 RabbitMQ 4.3 有哪些值得了解的新能力？

截至 2026-07-04，RabbitMQ 4.3 官方文档中，quorum queues 除了复制和高可用外，还提到一些更高级的能力，例如：

- poison message handling。
- at-least-once dead-lettering。
- delayed retry。
- consumer timeouts。
- strict message priority。

这些不是初级面试必须背的内容，但中高级面试中可以作为加分项。回答时要强调：是否启用取决于版本、业务目标和运维成本。

### 16.4 amqp091-go 版本选择要注意什么？

优先使用 RabbitMQ 团队维护的 `github.com/rabbitmq/amqp091-go`，不要继续把已停止维护的旧 `streadway/amqp` 当首选。

同时要注意：

- 锁定 Go module 版本。
- 阅读当前版本的 pkg.go.dev 文档。
- 注意 Channel 并发安全说明。
- 关注 NotifyClose、NotifyPublish、NotifyReturn 等异步事件。
- 连接恢复能力和行为要以实际版本文档为准。

## 17. 推荐复习顺序

面试前按这个顺序复习：

1. RabbitMQ 核心模型。
2. Exchange 类型。
3. Go 发布和消费代码。
4. Ack/Nack/Prefetch。
5. Publisher Confirm/Mandatory。
6. Retry/DLQ。
7. 幂等。
8. Outbox。
9. 消息堆积排查。
10. RabbitMQ vs Kafka。
11. 项目表达。

## 18. 参考资料

- [RabbitMQ AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [RabbitMQ Go Tutorial](https://www.rabbitmq.com/tutorials/tutorial-one-go)
- [RabbitMQ Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms)
- [RabbitMQ Consumer Prefetch](https://www.rabbitmq.com/docs/consumer-prefetch)
- [RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [RabbitMQ Time-To-Live and Expiration](https://www.rabbitmq.com/docs/ttl)
- [RabbitMQ Reliability Guide](https://www.rabbitmq.com/docs/reliability)
- [RabbitMQ Monitoring](https://www.rabbitmq.com/docs/monitoring)
- [RabbitMQ Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [RabbitMQ Streams](https://www.rabbitmq.com/docs/streams)
- [rabbitmq/amqp091-go](https://github.com/rabbitmq/amqp091-go)
- [amqp091-go package docs](https://pkg.go.dev/github.com/rabbitmq/amqp091-go)
