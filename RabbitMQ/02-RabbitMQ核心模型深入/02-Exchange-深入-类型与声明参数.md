# 02. Exchange 深入：类型与声明参数

## 1. Exchange 的职责

Exchange 是 RabbitMQ 的消息路由入口。

生产者通常不会直接把消息发给 queue，而是发给 exchange：

```text
Producer -> Exchange -> Queue
```

Exchange 收到消息后，会根据：

- exchange type
- routing key
- binding key
- headers 或 arguments

决定消息应该进入哪些队列。

## 2. Exchange 的常见类型

| 类型 | 路由规则 | 常见场景 |
| --- | --- | --- |
| direct | routing key 精确匹配 binding key | 任务类型、明确事件 |
| fanout | 广播到所有绑定队列 | 多服务订阅同一事件 |
| topic | routing key 和 binding key 按模式匹配 | 业务事件分类路由 |
| headers | 根据消息 headers 匹配 | 少用，了解即可 |

本阶段重点掌握：

```text
direct
fanout
topic
```

## 3. Direct Exchange

Direct exchange 使用精确匹配。

```text
routing key == binding key
```

示例：

```text
Exchange: stage2.direct
Queue: stage2.email.queue
Binding key: email.send
```

发布：

```text
routing key: email.send
```

消息进入队列。

发布：

```text
routing key: sms.send
```

消息不会进入队列。

Direct exchange 适合明确路由：

```text
email.send
sms.send
order.created
order.cancelled
```

## 4. Fanout Exchange

Fanout exchange 会把消息复制到所有绑定的队列。

```text
Exchange: stage2.fanout
  -> Queue: stage2.audit.queue
  -> Queue: stage2.analytics.queue
  -> Queue: stage2.notify.queue
```

生产者发一条消息，三个队列各收到一份。

Fanout exchange 通常不关心 routing key。

适合：

- 日志广播。
- 多系统监听同一事件。
- 用户注册、订单支付等事件通知多个服务。

## 5. Topic Exchange

Topic exchange 支持模式匹配。

Routing key 通常按点分隔：

```text
order.created
order.paid
order.payment.succeeded
user.registered
```

Binding key 支持：

| 通配符 | 含义 |
| --- | --- |
| `*` | 匹配一个单词 |
| `#` | 匹配零个或多个单词 |

示例：

```text
order.*  -> 匹配 order.created、order.paid
order.#  -> 匹配 order.created、order.payment.succeeded
*.created -> 匹配 order.created、user.created
```

Topic exchange 适合业务事件总线：

```text
order.events
user.events
payment.events
```

## 6. Exchange 声明参数

创建 exchange 时常见参数：

| 参数 | 含义 | 学习阶段建议 |
| --- | --- | --- |
| Name | exchange 名称 | 使用清晰业务命名 |
| Type | exchange 类型 | direct、fanout、topic |
| Durable | 是否持久化定义 | 业务 exchange 通常选 Yes |
| Auto delete | 无绑定时是否自动删除 | 业务 exchange 通常选 No |
| Internal | 是否只允许内部 exchange 使用 | 初学通常选 No |
| Arguments | 扩展参数 | 暂时不用 |

## 7. Durable 的含义

Durable exchange 表示：

```text
RabbitMQ 重启后，exchange 定义仍然存在。
```

但它不表示：

```text
消息一定不丢。
```

消息可靠性还需要 queue durable、message persistent、publisher confirm 等机制配合。后续阶段会深入。

学习阶段业务 exchange 建议：

```text
Durable: Yes
```

## 8. Auto Delete 的含义

Auto delete exchange 表示：

```text
当最后一个绑定被删除后，exchange 会自动删除。
```

业务系统中通常不希望 exchange 自动消失。

学习阶段建议：

```text
Auto delete: No
```

临时、测试、动态拓扑场景才考虑 auto-delete。

## 9. Internal 的含义

Internal exchange 不允许客户端直接发布消息到它。

它通常用于 exchange-to-exchange 的内部路由。

初学阶段不需要使用。

建议：

```text
Internal: No
```

## 10. 手动创建三个 exchange

在 Management UI 中创建：

```text
Exchanges -> Add a new exchange
```

### Direct

```text
Name: stage2.direct
Type: direct
Durability: Durable
Auto delete: No
Internal: No
```

### Fanout

```text
Name: stage2.fanout
Type: fanout
Durability: Durable
Auto delete: No
Internal: No
```

### Topic

```text
Name: stage2.topic
Type: topic
Durability: Durable
Auto delete: No
Internal: No
```

## 11. 用命令检查

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_exchanges name type durable auto_delete internal
```

你应该能看到：

```text
stage2.direct direct true false false
stage2.fanout fanout true false false
stage2.topic topic true false false
```

## 12. Exchange 选择建议

### 任务队列

如果是明确任务：

```text
发送邮件
发送短信
生成报表
```

可以用 direct：

```text
stage2.task.direct
```

### 事件广播

如果一个事件多个系统都要知道：

```text
user.registered
order.paid
```

可以用 fanout 或 topic。

### 业务事件总线

如果事件类型很多，并且需要按模式订阅：

```text
order.created
order.paid
order.cancelled
```

优先考虑 topic。

## 13. 本节小结

你要记住：

- exchange 负责路由，不负责长期存储消息。
- direct 精确匹配。
- fanout 广播。
- topic 模式匹配。
- 业务 exchange 通常 durable、非 auto-delete、非 internal。

下一节深入 queue。

