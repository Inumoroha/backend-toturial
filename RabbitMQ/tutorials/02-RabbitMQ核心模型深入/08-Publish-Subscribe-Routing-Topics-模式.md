# 08. Publish/Subscribe、Routing、Topics 模式

## 1. 本节目标

这一节把三种常见消息分发模式放在一起比较：

```text
Publish/Subscribe -> 广播
Routing           -> 按 key 精确分发
Topics            -> 按模式分发
```

这些模式是 RabbitMQ 官方教程中的核心基础，也是后续 Go 实战的主要代码结构来源。

## 2. Publish/Subscribe：发布订阅

发布订阅的核心是：

```text
一条消息，多个订阅者都收到一份。
```

拓扑：

```text
Producer -> fanout exchange
              -> queue A -> consumer A
              -> queue B -> consumer B
              -> queue C -> consumer C
```

使用场景：

- 用户注册后通知多个系统。
- 支付成功后通知订单、积分、通知、分析服务。
- 日志广播。

## 3. Publish/Subscribe 手动实验

创建 exchange：

```text
Name: stage2.events.fanout
Type: fanout
Durability: Durable
```

创建队列：

```text
stage2.events.email.queue
stage2.events.points.queue
stage2.events.analytics.queue
```

绑定：

```text
stage2.events.fanout -> stage2.events.email.queue
stage2.events.fanout -> stage2.events.points.queue
stage2.events.fanout -> stage2.events.analytics.queue
```

发布：

```text
exchange: stage2.events.fanout
routing key: 留空
payload:
{"event_type":"user.registered","user_id":1001}
```

预期：

```text
三个队列各收到一条消息。
```

## 4. Routing：按 key 精确分发

Routing 模式通常使用 direct exchange。

核心是：

```text
不同 routing key 进入不同队列。
```

拓扑：

```text
Producer -> direct exchange
              -- email.send --> email queue
              -- sms.send   --> sms queue
```

使用场景：

- 不同通知渠道。
- 不同任务类型。
- 明确的业务事件分类。

## 5. Routing 手动实验

创建 exchange：

```text
Name: stage2.notify.direct
Type: direct
Durability: Durable
```

创建队列：

```text
stage2.notify.email.queue
stage2.notify.sms.queue
```

绑定：

```text
stage2.notify.direct -- email.send --> stage2.notify.email.queue
stage2.notify.direct -- sms.send --> stage2.notify.sms.queue
```

发布：

```text
routing key: email.send
payload: {"channel":"email","to":"a@example.com"}
```

预期：

```text
stage2.notify.email.queue 收到消息
stage2.notify.sms.queue 不收到
```

发布：

```text
routing key: sms.send
payload: {"channel":"sms","phone":"13800000000"}
```

预期：

```text
stage2.notify.sms.queue 收到消息
stage2.notify.email.queue 不收到
```

## 6. Topics：按模式分发

Topics 模式通常使用 topic exchange。

核心是：

```text
消费者可以按业务模式订阅消息。
```

例如：

```text
order.#
*.created
payment.*
```

使用场景：

- 业务事件总线。
- 多类型事件订阅。
- 按业务域或事件类型过滤。

## 7. Topics 手动实验

创建 exchange：

```text
Name: stage2.business.topic
Type: topic
Durability: Durable
```

创建队列：

```text
stage2.business.order.queue
stage2.business.created.queue
stage2.business.payment.queue
```

绑定：

```text
stage2.business.topic -- order.# --> stage2.business.order.queue
stage2.business.topic -- *.created --> stage2.business.created.queue
stage2.business.topic -- payment.* --> stage2.business.payment.queue
```

发布：

```text
routing key: order.created
payload: {"event_type":"order.created","order_id":1001}
```

预期进入：

```text
stage2.business.order.queue
stage2.business.created.queue
```

发布：

```text
routing key: payment.succeeded
payload: {"event_type":"payment.succeeded","order_id":1001}
```

预期进入：

```text
stage2.business.payment.queue
```

## 8. 三种模式对比

| 模式 | Exchange | 特点 | 适合 |
| --- | --- | --- | --- |
| Publish/Subscribe | fanout | 所有绑定队列都收到 | 广播事件 |
| Routing | direct | 精确匹配 routing key | 明确任务或事件 |
| Topics | topic | 通配符模式匹配 | 业务事件总线 |

## 9. 和 Work Queue 的区别

Work Queue：

```text
一条任务 -> 一个队列 -> 多个 worker 中的一个处理
```

Publish/Subscribe：

```text
一条事件 -> 多个队列 -> 多个服务都处理
```

Routing / Topics：

```text
一条消息 -> 根据 key 或模式进入一个或多个队列
```

设计时先问：

```text
这条消息是任务，还是事件？
```

任务通常只需要处理一次。

事件通常可能有多个订阅者。

## 10. 本节小结

你要记住：

- 发布订阅适合广播事件。
- Routing 适合精确分类。
- Topics 适合模式订阅。
- Work Queue 是任务分发，不是广播。
- 一条消息进入几个队列，取决于 exchange 类型和 binding 设计。

下一节学习消息拓扑设计和命名规范。

