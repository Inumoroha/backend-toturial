# 04. Binding 与 Routing Key 深入

## 1. Binding 的本质

Binding 是 exchange 和 queue 之间的规则。

```text
Exchange -- Binding --> Queue
```

更准确地说：

```text
Exchange -- binding key / arguments --> Queue
```

它表达的是：

```text
这个队列想从这个 exchange 接收哪些消息。
```

## 2. Routing Key 的本质

Routing key 是生产者发布消息时附带的路由信息。

例如：

```text
email.send
sms.send
order.created
order.paid
user.registered
```

Exchange 收到消息后，会根据 exchange 类型来解释 routing key。

## 3. Direct 中的匹配

Direct exchange 的规则：

```text
routing key 精确匹配 binding key
```

示例：

```text
Exchange: stage2.direct
Queue: stage2.email.queue
Binding key: email.send
```

命中：

```text
routing key: email.send
```

不命中：

```text
routing key: email
routing key: email.send.high
routing key: sms.send
```

## 4. 一个 routing key 可以命中多个队列

Direct exchange 不代表只能一对一。

如果多个队列使用相同 binding key，都可以收到消息。

```text
stage2.direct -- order.created --> stage2.inventory.queue
stage2.direct -- order.created --> stage2.coupon.queue
stage2.direct -- order.created --> stage2.audit.queue
```

发布：

```text
routing key: order.created
```

三个队列都会收到一份消息。

所以 direct 可以一对一，也可以一对多，关键看 binding。

## 5. Fanout 中的匹配

Fanout exchange 通常忽略 routing key。

只要队列绑定到了 fanout exchange，就会收到消息。

```text
stage2.fanout -> stage2.audit.queue
stage2.fanout -> stage2.analytics.queue
stage2.fanout -> stage2.notify.queue
```

发布任意 routing key：

```text
hello
order.created
anything
```

这些队列通常都会收到消息。

## 6. Topic 中的匹配

Topic exchange 使用模式匹配。

Routing key 按点分隔：

```text
order.created
order.payment.succeeded
user.registered
```

Binding key 支持：

```text
*  匹配一个单词
#  匹配零个或多个单词
```

### 6.1 `*` 示例

```text
binding key: order.*
```

匹配：

```text
order.created
order.paid
```

不匹配：

```text
order.payment.succeeded
```

因为 `*` 只匹配一个单词。

### 6.2 `#` 示例

```text
binding key: order.#
```

匹配：

```text
order
order.created
order.payment.succeeded
```

`#` 可以匹配零个或多个单词。

### 6.3 混合示例

```text
binding key: *.payment.#
```

可能匹配：

```text
order.payment.succeeded
user.payment.failed
```

## 7. 手动实验：Direct 多队列绑定

创建 exchange：

```text
Name: stage2.direct
Type: direct
```

创建队列：

```text
stage2.inventory.queue
stage2.coupon.queue
stage2.audit.queue
```

绑定：

```text
stage2.direct -- order.created --> stage2.inventory.queue
stage2.direct -- order.created --> stage2.coupon.queue
stage2.direct -- order.created --> stage2.audit.queue
```

发布：

```text
exchange: stage2.direct
routing key: order.created
payload: direct multi queue test
```

预期：

```text
三个队列各收到一条消息。
```

## 8. 手动实验：Topic 模式匹配

创建 exchange：

```text
Name: stage2.topic
Type: topic
```

创建队列和绑定：

```text
stage2.order.all.queue       binding key: order.#
stage2.order.onelevel.queue  binding key: order.*
stage2.created.all.queue     binding key: *.created
```

发布：

```text
routing key: order.created
```

预期进入：

```text
stage2.order.all.queue
stage2.order.onelevel.queue
stage2.created.all.queue
```

发布：

```text
routing key: order.payment.succeeded
```

预期进入：

```text
stage2.order.all.queue
```

发布：

```text
routing key: user.created
```

预期进入：

```text
stage2.created.all.queue
```

## 9. 用命令查看 binding

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_bindings
```

你会看到 source、destination、routing key 等信息。

重点看：

```text
source exchange
destination queue
routing key
```

## 10. Routing Key 命名建议

推荐使用事件风格：

```text
order.created
order.paid
order.cancelled
user.registered
notification.email.send
```

不推荐：

```text
create
paid
test
abc
1
```

命名要表达业务含义。

## 11. 本节小结

你要记住：

- Binding 决定 queue 从 exchange 接收哪些消息。
- Routing key 是消息路由信息。
- Direct 是精确匹配。
- Fanout 通常忽略 routing key。
- Topic 支持 `*` 和 `#` 通配符。
- 一个消息可以进入多个队列。

下一节学习默认交换机。

