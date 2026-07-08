# 07. Direct、Fanout、Topic 交换机入门

## 1. 为什么要学 exchange 类型

Exchange 的核心职责是路由消息。

不同 exchange 类型代表不同路由规则：

```text
direct -> 精确匹配
fanout -> 广播
topic  -> 通配符匹配
```

这一节你不需要掌握所有细节，只需要通过手动实验理解三者差异。

## 2. Direct Exchange：精确路由

Direct exchange 根据 routing key 精确匹配 binding key。

示例：

```text
exchange: learning.direct
queue: learning.email.queue
binding key: email
```

发布消息：

```text
routing key: email
```

消息进入队列。

发布消息：

```text
routing key: sms
```

消息不会进入这个队列。

### 适合场景

- 指定任务类型。
- 指定业务动作。
- 明确的一对一或一对多路由。

例如：

```text
notification.email
notification.sms
order.created
order.cancelled
```

## 3. Fanout Exchange：广播

Fanout exchange 会把消息广播给所有绑定到它的队列，通常不关心 routing key。

示例：

```text
exchange: learning.fanout
  -> queue: learning.log.console.queue
  -> queue: learning.log.file.queue
  -> queue: learning.log.audit.queue
```

生产者发布一条消息到 `learning.fanout`，三个队列都会收到一份消息。

### 手动实验

创建 fanout exchange：

```text
Name: learning.fanout
Type: fanout
Durability: Durable
```

创建三个 queue：

```text
learning.fanout.a.queue
learning.fanout.b.queue
learning.fanout.c.queue
```

把三个 queue 都绑定到 `learning.fanout`。

然后发布一条消息：

```text
exchange: learning.fanout
routing key: 任意填写或留空
payload: fanout test
```

你会看到三个队列都增加一条消息。

### 适合场景

- 广播通知。
- 多个系统都要感知同一事件。
- 日志分发。

例如：

```text
user.registered
  -> email-service
  -> points-service
  -> analytics-service
```

## 4. Topic Exchange：通配符路由

Topic exchange 根据 routing key 和 binding key 的模式匹配。

常见 routing key：

```text
order.created
order.paid
order.cancelled
user.created
user.deleted
```

Topic binding key 支持两个通配符：

| 通配符 | 含义 |
| --- | --- |
| `*` | 匹配一个单词 |
| `#` | 匹配零个或多个单词 |

单词之间用点分隔：

```text
order.created
order.payment.succeeded
```

### 示例一：匹配所有订单一级事件

```text
binding key: order.*
```

可以匹配：

```text
order.created
order.paid
order.cancelled
```

不能匹配：

```text
order.payment.succeeded
```

因为 `*` 只匹配一个单词。

### 示例二：匹配所有订单事件

```text
binding key: order.#
```

可以匹配：

```text
order.created
order.paid
order.payment.succeeded
order.payment.failed
```

### 示例三：匹配所有 created 事件

```text
binding key: *.created
```

可以匹配：

```text
order.created
user.created
product.created
```

## 5. Topic 手动实验

创建 topic exchange：

```text
Name: learning.topic
Type: topic
Durability: Durable
```

创建队列：

```text
learning.order.all.queue
learning.order.created.queue
learning.created.all.queue
```

创建 binding：

```text
learning.topic -- order.# --> learning.order.all.queue
learning.topic -- order.created --> learning.order.created.queue
learning.topic -- *.created --> learning.created.all.queue
```

然后分别发布：

```text
routing key: order.created
```

预期进入：

```text
learning.order.all.queue
learning.order.created.queue
learning.created.all.queue
```

再发布：

```text
routing key: order.paid
```

预期进入：

```text
learning.order.all.queue
```

再发布：

```text
routing key: user.created
```

预期进入：

```text
learning.created.all.queue
```

## 6. 三种 exchange 对比

| 类型 | 路由方式 | 典型场景 |
| --- | --- | --- |
| direct | routing key 精确匹配 | 指定任务类型、明确事件 |
| fanout | 广播到所有绑定队列 | 多系统订阅同一事件 |
| topic | 通配符匹配 | 事件分类、业务域路由 |

初学建议：

- 任务队列先用 direct。
- 广播通知用 fanout。
- 业务事件路由用 topic。

## 7. 本节小结

你现在应该能解释：

- direct 是精确匹配。
- fanout 是广播。
- topic 是模式匹配。
- `*` 匹配一个单词。
- `#` 匹配零个或多个单词。

下一节学习 vhost、用户、权限和命名规范。

