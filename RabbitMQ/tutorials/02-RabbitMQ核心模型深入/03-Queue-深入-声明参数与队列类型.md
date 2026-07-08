# 03. Queue 深入：声明参数与队列类型

## 1. Queue 的职责

Queue 是消息真正等待消费的地方。

```text
Exchange -> Queue -> Consumer
```

Exchange 决定消息去哪，Queue 负责保存消息，Consumer 从 Queue 中取消息处理。

## 2. Queue 的常见声明参数

创建 queue 时常见参数：

| 参数 | 含义 | 学习阶段建议 |
| --- | --- | --- |
| Name | 队列名称 | 使用清晰业务命名 |
| Type | 队列类型 | 初学使用 Classic |
| Durable | 是否持久化队列定义 | 业务队列通常 Yes |
| Exclusive | 是否只允许声明它的连接使用 | 业务队列通常 No |
| Auto delete | 最后一个 consumer 取消后是否自动删除 | 业务队列通常 No |
| Arguments | 扩展参数 | 后续学习 TTL、DLX、长度限制等 |

## 3. Durable Queue

Durable queue 表示：

```text
RabbitMQ 重启后，队列定义仍然存在。
```

但它不等于消息一定不丢。

要让消息更可靠，通常需要组合：

```text
durable exchange
durable queue
persistent message
publisher confirm
manual ack
```

这一阶段先记住：

```text
业务队列通常 durable。
```

## 4. Exclusive Queue

Exclusive queue 表示：

```text
这个队列只能被声明它的连接使用。
连接关闭后，队列通常会被删除。
```

适合：

- 临时回调队列。
- RPC reply queue。
- 临时订阅。
- 测试和实验。

不适合：

- 订单队列。
- 支付队列。
- 通知任务队列。
- 任何长期业务队列。

业务队列建议：

```text
Exclusive: No
```

## 5. Auto Delete Queue

Auto delete queue 表示：

```text
至少有过一个 consumer，且最后一个 consumer 取消后，队列会自动删除。
```

适合：

- 临时订阅。
- 测试队列。
- 客户端生命周期绑定的队列。

不适合长期业务队列。

业务队列建议：

```text
Auto delete: No
```

## 6. Queue Type

RabbitMQ 常见队列类型：

| 类型 | 特点 | 适用场景 |
| --- | --- | --- |
| Classic Queue | 传统队列，入门和一般任务常用 | 学习、普通任务队列 |
| Quorum Queue | 基于复制日志，面向数据安全和高可用 | 关键业务消息 |
| Stream | 日志式存储，可重复读取，偏事件流 | 高吞吐事件流、回放 |

本阶段实验统一使用：

```text
Classic Queue
```

Quorum Queue 和 Stream 后续高可用阶段再深入。

## 7. 手动创建业务队列

在 UI 中进入：

```text
Queues and Streams -> Add a new queue
```

创建邮件队列：

```text
Type: Classic
Name: stage2.email.queue
Durability: Durable
Exclusive: No
Auto delete: No
```

创建短信队列：

```text
Type: Classic
Name: stage2.sms.queue
Durability: Durable
Exclusive: No
Auto delete: No
```

创建订单事件队列：

```text
Type: Classic
Name: stage2.order.created.queue
Durability: Durable
Exclusive: No
Auto delete: No
```

## 8. 用命令检查队列

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name type durable exclusive auto_delete messages_ready messages_unacknowledged consumers
```

你应该能看到类似：

```text
stage2.email.queue classic true false false 0 0 0
stage2.sms.queue classic true false false 0 0 0
stage2.order.created.queue classic true false false 0 0 0
```

如果你的 RabbitMQ 版本不显示 `type` 字段，可以改用：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name durable exclusive auto_delete messages_ready messages_unacknowledged consumers
```

## 9. Ready、Unacked、Consumers

Queue 页面里最重要的三个指标：

| 指标 | 含义 |
| --- | --- |
| Ready | 等待被投递的消息数量 |
| Unacked | 已投递给消费者但还没确认的消息数量 |
| Consumers | 当前消费者数量 |

### Ready 很高

说明：

```text
消息堆在队列里，消费者消费不过来或没有消费者。
```

### Unacked 很高

说明：

```text
消息已经交给消费者，但消费者还没 ack。
```

可能是消费者处理慢、卡住或 prefetch 过大。

## 10. Queue 命名建议

Queue 通常应该按消费者或业务用途命名。

好名字：

```text
order.created.inventory.queue
order.paid.points.queue
user.registered.email.queue
notification.email.worker.queue
```

坏名字：

```text
queue1
test
data
event
mq
```

一个实用原则：

```text
看到 queue 名字，就应该大概知道谁在消费、消费什么。
```

## 11. Queue 的归属

真实项目中，queue 更偏消费者所有。

例如：

```text
Exchange: order.events
Routing key: order.created

Queue:
order.created.inventory.queue -> inventory-service 消费
order.created.coupon.queue    -> coupon-service 消费
order.created.audit.queue     -> audit-service 消费
```

生产者只发布 `order.created`，不应该强依赖所有消费者队列。

## 12. 本节小结

你要记住：

- Queue 负责保存消息，Consumer 从 Queue 消费。
- 业务队列通常 durable、非 exclusive、非 auto-delete。
- Exclusive 和 auto-delete 更适合临时队列。
- 初学和普通任务先用 Classic Queue。
- Queue 名称要体现消费者或业务用途。

下一节深入 binding 和 routing key。

