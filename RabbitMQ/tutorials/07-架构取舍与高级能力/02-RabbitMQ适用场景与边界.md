# 02. RabbitMQ 适用场景与边界

## 1. RabbitMQ 的核心优势

RabbitMQ 的强项是业务消息。

它特别适合：

- 任务队列。
- 低延迟异步处理。
- 复杂路由。
- 多消费者订阅事件。
- 可靠业务投递。
- 削峰填谷。
- Go 后端 worker 系统。

一句话：

```text
RabbitMQ 很适合后端服务之间的业务消息分发。
```

## 2. 适合场景一：任务队列

例如：

- 发送邮件。
- 发送短信。
- 生成报表。
- 图片压缩。
- 视频转码。
- 数据导入。

拓扑：

```text
producer -> task exchange -> task queue -> workers
```

特点：

- 每条任务通常只处理一次。
- 多个 worker 水平扩展。
- manual ack 保证失败可重投。
- retry + DLQ 处理失败。

## 3. 适合场景二：业务事件通知

例如：

```text
order.created
order.paid
user.registered
payment.succeeded
```

拓扑：

```text
order-service -> order.events(topic)
  -> inventory queue
  -> points queue
  -> notification queue
  -> audit queue
```

RabbitMQ 的 topic exchange 很适合按业务事件路由。

## 4. 适合场景三：复杂路由

RabbitMQ exchange 类型丰富：

- direct：精确匹配。
- fanout：广播。
- topic：模式匹配。
- headers：基于 headers。

如果你的业务需要：

```text
同一条消息按 routing key 进入不同队列
某些服务订阅 order.#
某些服务只订阅 order.paid
某些事件需要广播
```

RabbitMQ 很适合。

## 5. 适合场景四：低延迟业务异步

RabbitMQ 常用于：

```text
HTTP API 快速返回
后台 worker 异步处理
```

例如：

```text
用户注册 -> 发布 user.registered -> 立即返回
邮件、积分、分析异步处理
```

这种业务通常更看重：

- 低延迟。
- 路由灵活。
- 失败重试。
- 业务可恢复。

## 6. 适合场景五：可靠工作流的一部分

RabbitMQ 可以配合：

- publisher confirm。
- manual ack。
- durable queue。
- persistent message。
- retry queue。
- DLQ。
- Transactional Outbox。
- 幂等消费者。

构建可靠业务链路。

注意：

```text
RabbitMQ 是可靠链路的一部分，不是全部。
```

## 7. 不适合场景一：大规模事件长期回放

如果你的核心需求是：

- 保存海量事件。
- 多个消费组长期回放。
- 从任意 offset 重读。
- 做实时数仓和数据管道。

Kafka 通常更合适。

普通 RabbitMQ queue 的设计目标不是长期日志存储。

RabbitMQ Streams 可以覆盖一部分流式场景，但它和普通 queue 是不同模型。

## 8. 不适合场景二：超大消息体

不要把大文件、大图片、大视频直接塞进 RabbitMQ。

推荐：

```text
文件放对象存储
消息里只放 file_id / object_key
```

例如：

```json
{
  "file_id": "file-001",
  "object_key": "uploads/2026/07/04/a.mp4"
}
```

## 9. 不适合场景三：强同步查询

例如：

```text
查询用户详情
查询商品库存
登录校验密码
```

用户需要立即拿到结果，通常应该用 HTTP/gRPC 或数据库查询。

不要为了异步而异步。

## 10. 不适合场景四：团队没有运维能力

RabbitMQ 需要：

- 监控。
- 告警。
- DLQ 处理。
- 权限治理。
- 容量规划。
- 故障演练。

如果团队完全没有这些能力，而任务量很小，数据库任务表可能更合适。

## 11. 判断口诀

适合 RabbitMQ：

```text
业务消息
异步处理
复杂路由
任务分发
低延迟
失败可恢复
```

谨慎使用 RabbitMQ：

```text
海量日志
长期回放
强同步查询
大文件传输
极简小任务
团队无运维能力
```

## 12. 本节小结

RabbitMQ 的定位：

```text
业务消息代理和任务队列系统。
```

它最强的是：

```text
exchange + queue + binding 带来的灵活路由，以及成熟的消费确认和失败处理机制。
```

下一节对比 RabbitMQ 和 Kafka。

