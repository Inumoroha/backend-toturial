# 07. 任务队列、事件驱动、RPC 的边界

## 1. 为什么要区分通信模式

很多 RabbitMQ 设计混乱，是因为没有区分：

- task。
- event。
- RPC。
- 同步 HTTP/gRPC。

这些模式的语义不同。

如果语义混乱，系统会越来越难维护。

## 2. 任务队列

Task 表示：

```text
请执行一个动作。
```

例如：

```text
notification.email.send
image.compress
report.generate
```

特点：

- 一条任务通常只需要一个 worker 执行。
- 可以重试。
- 可以进入 DLQ。
- 适合 Work Queue。

## 3. 事件驱动

Event 表示：

```text
某件事情已经发生。
```

例如：

```text
order.created
order.paid
user.registered
payment.succeeded
```

特点：

- 多个消费者可能感兴趣。
- 生产者不关心谁消费。
- 适合 topic/fanout。
- 更解耦。

## 4. RPC

RPC 表示：

```text
我发起请求，并等待响应。
```

RabbitMQ 可以实现 RPC：

- request queue。
- reply_to。
- correlation_id。

但不要默认用 RabbitMQ 做 RPC。

如果调用方必须立即拿到结果，HTTP/gRPC 通常更直接。

## 5. 同步调用

同步 HTTP/gRPC 适合：

- 查询。
- 强实时校验。
- 必须立即得到结果。
- 简单调用链。

例如：

```text
登录校验密码
查询商品详情
获取用户资料
```

不要把所有同步场景都改成 MQ。

## 6. 判断方式

问三个问题：

### 6.1 调用方是否必须立即知道结果？

是：

```text
优先同步 HTTP/gRPC 或 RPC。
```

否：

```text
可以异步消息。
```

### 6.2 消息是命令还是事实？

命令：

```text
task
```

事实：

```text
event
```

### 6.3 是否有多个消费者都关心？

是：

```text
event + topic/fanout
```

否：

```text
task queue 或同步调用
```

## 7. 常见错误

### 错误一：事件写成命令

不推荐：

```text
call_inventory_service
send_order_sms
```

推荐：

```text
order.created
order.paid
```

### 错误二：任务写成事件

不推荐：

```text
email.send.created
```

推荐：

```text
notification.email.send
```

### 错误三：用 MQ 做所有查询

查询类请求通常不适合 MQ。

除非你明确在做异步查询或离线处理。

## 8. 模式对照表

| 需求 | 推荐模式 |
| --- | --- |
| 发送邮件 | Task Queue |
| 订单支付成功通知多个系统 | Event Driven |
| 查询用户详情 | HTTP/gRPC |
| 需要请求响应但必须经过消息系统 | RabbitMQ RPC，谨慎使用 |
| 用户行为日志分析 | Kafka / Stream |
| 简单低频后台任务 | 数据库任务表 |

## 9. 本节小结

你要记住：

- task 是“做事”。
- event 是“事实”。
- RPC 是“请求响应”。
- 查询通常用同步调用。
- RabbitMQ 很强，但不该替代所有通信方式。

下一节学习吞吐、延迟、顺序、回放和大消息取舍。

