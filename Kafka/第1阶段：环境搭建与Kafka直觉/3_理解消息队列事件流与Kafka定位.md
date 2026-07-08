# 3. 理解消息队列、事件流与 Kafka 定位

本节目标：建立 Kafka 的整体直觉，理解它为什么适合 Go 后端中的异步、削峰、解耦和事件驱动场景。

很多初学者会把 Kafka 简单理解成“消息队列”。这个说法不算错，但不完整。Kafka 更准确地说是一个分布式事件流平台。它既能像消息队列一样让多个消费者分工处理任务，也能像事件日志一样长期保存事件，让不同系统按自己的进度读取。

---

## 一、先从一个后端场景开始

假设你正在写一个电商后端。

用户创建订单后，系统可能要做很多事：

```text
创建订单。
扣减库存。
创建支付单。
发送短信。
发优惠券。
写行为日志。
更新搜索索引。
更新数据看板。
```

如果全部在 HTTP 请求里同步完成，请求链路可能变成：

```text
Client
  -> order-service
  -> inventory-service
  -> payment-service
  -> coupon-service
  -> notification-service
  -> analytics-service
```

问题很明显：

- 链路太长。
- 任意下游变慢都会拖慢创建订单接口。
- 新增一个下游就要改订单服务。
- 某个非核心下游故障会影响核心业务。

Kafka 的常见做法是：

```text
order-service 只负责创建订单并发布 order.created

inventory-service      订阅 order.created
coupon-service         订阅 order.created
notification-service   订阅 order.created
analytics-service      订阅 order.created
```

订单服务发布一个“事实”：

```text
订单已经创建
```

下游服务自己决定要不要关心这个事实。

---

## 二、消息队列解决什么问题

消息队列最常见的三个价值：

### 1. 异步

同步调用：

```text
创建订单 -> 发短信 -> 返回用户
```

发短信慢，用户就等得久。

异步：

```text
创建订单 -> 写 Kafka -> 返回用户
短信服务稍后消费消息
```

用户不需要等短信真的发出去。

### 2. 削峰

秒杀时请求瞬间涌入。

如果所有请求直接打到数据库，数据库可能被压垮。

使用 Kafka 后：

```text
请求高峰 -> Kafka 暂存消息 -> consumer 按能力慢慢处理
```

Kafka 像一个缓冲层。

### 3. 解耦

订单服务不需要知道哪些服务关心订单创建事件。

新增 `analytics-service` 时，只要它订阅 `order.created`，不需要改订单服务。

---

## 三、Kafka 与普通队列的区别

普通队列常见模型：

```text
producer -> queue -> consumer
```

消息被 consumer 处理后，通常就从队列里删除。

Kafka 的模型更像：

```text
producer -> topic partition log -> consumer group offset
```

消息写入 Kafka 后，会按 topic 的保留策略保存一段时间。consumer 是否读过，不直接决定消息是否删除。

这带来一个重要能力：

```text
多个 consumer group 可以独立读取同一份事件日志
```

例如：

```text
order.created
  -> inventory-service
  -> notification-service
  -> analytics-service
```

每个 group 都有自己的 offset。

---

## 四、Kafka 更像“事件日志”

Kafka 的核心存储模型是 append-only log，也就是追加写日志。

可以想象成：

```text
partition-0:
  offset 0: order-1 created
  offset 1: order-2 created
  offset 2: order-3 created
```

consumer 不会把消息“拿走”，而是记录自己读到了哪里。

这和数据库表有点不同：

- 数据库表关注当前状态。
- Kafka topic 更关注事件流。

例如订单表里可能只有：

```text
order_id = 1001, status = paid
```

但事件流里可以有：

```text
order.created
inventory.deducted
payment.succeeded
notification.sent
```

事件流记录了状态变化过程。

---

## 五、Kafka 适合什么场景

### 1. 业务事件

例如：

- `order.created`
- `payment.succeeded`
- `inventory.deducted`
- `user.registered`

### 2. 日志采集

例如：

- HTTP 访问日志。
- 用户行为日志。
- App 埋点。

### 3. 数据同步

例如：

- MySQL/PostgreSQL 变更同步到 Elasticsearch。
- 订单数据同步到 ClickHouse。
- 用户数据同步到推荐系统。

### 4. 后台任务

例如：

- 异步生成报表。
- 异步发送通知。
- 延迟处理非核心任务。

---

## 六、Kafka 不适合什么场景

Kafka 很强，但不是万能的。

### 1. 不适合做强实时 RPC

如果你需要立即拿到库存服务返回值，Kafka 不如 HTTP/gRPC 直接。

Kafka 更适合异步事件。

### 2. 不适合保存大文件

不要把图片、视频、PDF 直接塞进 Kafka。

更合理的做法：

```text
文件放对象存储。
Kafka 消息里只放文件 URL 或对象 key。
```

### 3. 不适合替代数据库

Kafka 可以保存消息，但它不是通用查询数据库。

你不能像查 PostgreSQL 那样随意：

```sql
SELECT * FROM orders WHERE user_id = 'u1'
```

Kafka 适合按顺序读取事件，不适合复杂条件查询。

---

## 七、Go 后端中 Kafka 的常见位置

一个 Go 后端服务可能同时有 producer 和 consumer。

例如 `order-service`：

```text
HTTP Handler
  -> Order Service
  -> PostgreSQL
  -> Kafka Producer 发布 order.created
```

例如 `inventory-service`：

```text
Kafka Consumer 订阅 order.created
  -> Inventory Handler
  -> PostgreSQL 扣库存
  -> Kafka Producer 发布 inventory.deducted
```

所以 Kafka 在 Go 项目里通常会有一个基础包：

```text
internal/kafka/
  producer.go
  consumer.go
  message.go
  retry.go
  dlq.go
```

业务代码不应该到处直接调用第三方 Kafka 客户端 API，而应该通过这个基础包统一处理。

---

## 八、事件命名建议

Kafka topic 命名建议表达“已经发生的事实”。

推荐：

```text
order.created
order.cancelled
payment.succeeded
inventory.deducted
user.registered
```

不推荐：

```text
create_order
send_sms
call_inventory
process_payment
```

为什么？

事件应该告诉下游：

```text
发生了什么
```

而不是命令下游：

```text
你应该做什么
```

这样下游服务可以自由决定如何响应这个事件。

---

## 九、常见误解

### 1. Kafka 能保证业务只执行一次

不能。

Kafka 可以提供一些精确一次语义，但业务系统仍然要处理重复消费。

例如发优惠券时，consumer 必须用唯一约束或幂等表防止重复发券。

### 2. 消息被消费后就消失

Kafka 中消息是否删除，主要取决于 topic 的保留策略，而不是某个 consumer 是否读过。

### 3. consumer 越多越快

不一定。

同一个 consumer group 的并行度上限受 partition 数量限制。

### 4. Kafka 可以替代所有异步任务系统

不一定。

复杂延迟任务、定时任务、工作流编排，可能需要专门的任务系统或工作流引擎。

---

## 十、本节练习

1. 写出你熟悉的一个后端业务流程。
2. 标出其中哪些步骤可以异步。
3. 为这些异步步骤设计事件名称。
4. 判断这些事件应该由哪个服务发布。
5. 判断哪些服务会消费这些事件。
6. 思考：如果某个 consumer 停止 1 小时，会不会影响 producer 写入？

---

## 十一、本节小结

- Kafka 不只是普通消息队列，更像分布式事件日志。
- Kafka 适合异步、削峰、解耦、事件流和数据同步。
- Kafka 消息被消费后不会立即删除。
- 多个 consumer group 可以独立消费同一个 topic。
- Go 后端中，Kafka producer 常在业务服务里发布事件。
- Go 后端中，Kafka consumer 常是后台 worker。
- 事件命名应表达已经发生的事实。
- Kafka 不能替代业务幂等设计。

---

## 十二、业务拆分示例

以创建订单为例：

```text
同步完成：
  校验请求
  写订单数据库
  返回订单创建成功

异步完成：
  扣减库存
  发送通知
  写入分析系统
```

可以发布事件：

```text
order.created
```

库存、通知、分析服务分别消费这个事件。这样 order-service 不需要同步等待所有下游完成。

---

## 十三、判断是否适合 Kafka

适合：

- 下游可以稍后处理。
- 多个系统需要响应同一事件。
- 需要削峰。
- 需要保留事件用于重放。

不适合：

- 用户必须立即拿到强一致结果。
- 逻辑很简单，同步调用更清楚。
- 团队还没有能力处理重复消费和失败消息。

---

## 二十、用订单场景做判断

练习时可以拿订单系统做判断：

```text
创建订单后扣库存：可以用事件，但要接受最终一致。
用户支付页面立即显示支付结果：通常需要同步查询或同步确认。
订单创建后发送短信：非常适合异步事件。
后台生成运营报表：适合消费订单事件流。
```

这类判断题能帮助你建立边界感：Kafka 不是为了替代所有接口调用，而是为了把不必同步完成的事情拆出去。
