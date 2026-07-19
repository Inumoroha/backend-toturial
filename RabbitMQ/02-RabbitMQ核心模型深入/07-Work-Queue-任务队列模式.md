# 07. Work Queue 任务队列模式

## 1. Work Queue 解决什么问题

Work Queue，也叫任务队列。

它解决的是：

```text
一批任务需要多个 worker 共同处理。
```

典型场景：

- 发送邮件。
- 生成报表。
- 图片压缩。
- 视频转码。
- 数据导入。
- 批量同步。

## 2. 基础结构

```text
Producer -> task queue -> worker-1
                       -> worker-2
                       -> worker-3
```

特点：

- 多个 worker 监听同一个 queue。
- 每条消息通常只会被其中一个 worker 处理。
- 增加 worker 数量可以提升处理能力。

## 3. Work Queue 和广播的区别

Work Queue：

```text
一条任务 -> 一个 worker 处理
```

Publish/Subscribe：

```text
一条事件 -> 多个队列各收到一份
```

不要混淆这两个模式。

例如发送 1000 封邮件：

```text
适合 Work Queue
```

因为每封邮件只需要一个 worker 发送一次。

例如订单支付成功通知多个系统：

```text
适合 Publish/Subscribe
```

因为订单、积分、通知、分析系统都要知道。

## 4. Work Queue 拓扑设计

可以使用 direct exchange：

```text
Exchange: stage2.task.direct
Type: direct

Queue: stage2.email.task.queue
Binding key: email.send
```

消息流：

```text
producer -- email.send --> stage2.task.direct
stage2.task.direct -- email.send --> stage2.email.task.queue
worker 从 stage2.email.task.queue 消费
```

也可以用默认 exchange 直接投递到队列：

```text
exchange: ""
routing key: stage2.email.task.queue
```

学习和小项目可以用默认 exchange，业务系统更推荐显式 exchange。

## 5. 手动创建 Work Queue

创建 exchange：

```text
Name: stage2.task.direct
Type: direct
Durability: Durable
```

创建 queue：

```text
Name: stage2.email.task.queue
Type: Classic
Durability: Durable
Exclusive: No
Auto delete: No
```

绑定：

```text
stage2.task.direct -- email.send --> stage2.email.task.queue
```

## 6. 发布多个任务

进入：

```text
Exchanges -> stage2.task.direct -> Publish message
```

连续发布 3 条消息。

第一条：

```text
Routing key: email.send
Payload:
{"task_id":"email-001","to":"a@example.com"}
```

第二条：

```text
Routing key: email.send
Payload:
{"task_id":"email-002","to":"b@example.com"}
```

第三条：

```text
Routing key: email.send
Payload:
{"task_id":"email-003","to":"c@example.com"}
```

查看队列：

```text
Queues and Streams -> stage2.email.task.queue
```

预期：

```text
Ready: 3
```

## 7. 手动模拟消费

在 queue 页面使用：

```text
Get messages
Messages: 1
Ack mode: Ack message requeue false
```

每点击一次，Ready 数量减少 1。

这只是手动模拟。真正的 worker 会持续监听队列并处理消息。

## 8. 多 worker 的意义

假设每条消息处理 1 秒。

一个 worker：

```text
1000 条消息约 1000 秒
```

五个 worker：

```text
理想情况下约 200 秒
```

实际效果还会受数据库、外部 API、网络、ack、prefetch 等因素影响。

## 9. Work Queue 后续还要学什么

真正可靠的 Work Queue 需要：

- manual ack
- prefetch
- 失败重试
- 死信队列
- 幂等处理
- 优雅退出

这些内容后续阶段会逐步展开。

本阶段只要理解拓扑模式：

```text
一个队列，多个 worker，共同分担任务。
```

## 10. Work Queue 适合和不适合

适合：

- 每条消息只需要处理一次。
- 可以水平扩展 worker。
- 任务处理时间可能较长。

不适合：

- 一条消息需要多个服务都收到。
- 强实时同步查询。
- 需要复杂事件订阅的系统。

## 11. 本节小结

你要记住：

- Work Queue 是任务分发模式。
- 多个 worker 共享同一个 queue。
- 一条任务通常只被一个 worker 处理。
- 它和广播模式不是一回事。

下一节学习 Publish/Subscribe、Routing、Topics 三种事件分发模式。

