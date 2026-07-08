# Kafka 模拟面试

这份文档用于训练真实面试表达。

它不是题库，而是模拟面试现场的对话节奏：

```text
面试官提问
-> 你的回答
-> 面试官追问
-> 更好的回答
```

建议使用方式：

1. 先只看“面试官提问”，自己口头回答。
2. 再看“你的回答”，判断哪里不够。
3. 最后看“更好的回答”，学习如何把答案讲完整。
4. 不要死背原文，要改成自己的项目表达。

面试中，Kafka 问题最忌讳只背概念。比较好的回答通常包含：

- 原理。
- 使用场景。
- 项目例子。
- 风险。
- 生产实践。
- Go 后端代码边界。

---

## 第一轮：为什么使用 Kafka

### 面试官提问

你为什么在项目里使用 Kafka？

### 你的回答

因为 Kafka 可以做异步消息队列，订单服务创建订单后，把消息发到 Kafka，然后库存服务去消费。

### 面试官追问

那为什么不用 HTTP 直接调用库存服务？引入 Kafka 不是更复杂吗？

### 更好的回答

我们引入 Kafka 主要是为了解耦订单服务和多个下游系统。

在订单创建场景里，订单服务创建订单后，不只有库存服务要处理，后面还可能有通知服务、积分服务、数据分析服务。如果全部用 HTTP 同步调用，订单接口会依赖多个下游服务的稳定性和响应时间。

使用 Kafka 后，订单服务只需要发布一个 `order.created` 事件。库存服务、通知服务、分析服务分别作为不同的 consumer group 消费同一份事件，互不影响。

但我不会说 Kafka 一定比 HTTP 好。Kafka 带来了最终一致性、重复消费、消息堆积、DLQ、监控告警等复杂度。只有当系统确实需要异步解耦、削峰填谷、多下游订阅或事件留存时，引入 Kafka 才比较合适。

我在项目里的设计是：

```text
订单服务写订单和 outbox。
后台 publisher 发送 order.created 到 Kafka。
库存服务消费后预占库存。
失败进入 retry 或 DLQ。
通过 consumer lag 和 DLQ 指标观察链路健康。
```

### 这一轮考察点

- 是否知道 Kafka 的业务价值。
- 是否能讲出不用 Kafka 的替代方案。
- 是否知道 Kafka 也会带来复杂度。
- 是否能结合订单项目说明。

---

## 第二轮：Kafka 和 RabbitMQ 的区别

### 面试官提问

Kafka 和 RabbitMQ 有什么区别？

### 你的回答

Kafka 吞吐更高，RabbitMQ 功能更多。Kafka 适合大数据，RabbitMQ 适合普通消息队列。

### 面试官追问

你这个回答有点泛。能不能从消息存储、消费模型和业务场景讲一下？

### 更好的回答

Kafka 更像一个分布式事件日志系统。消息按 topic 和 partition 追加写入，consumer 通过 offset 记录自己的消费进度。消息消费后不会立即删除，而是根据 retention 或 compaction 策略保留一段时间。所以多个 consumer group 可以重复消费同一份 topic。

RabbitMQ 更偏传统消息队列，强调 exchange、queue、routing key 等路由能力。消息通常被消费者确认后从队列中移除，更适合任务队列、复杂路由、请求异步处理等场景。

如果我的业务是订单事件流、用户行为日志、多个系统订阅同一事件、需要高吞吐和消息留存，我更倾向 Kafka。

如果业务更强调灵活路由、任务分发、延迟消息或较轻量的消息队列模型，我会考虑 RabbitMQ。

所以我不会简单说谁更好，而是看业务需要：

```text
事件流、多订阅、高吞吐、可重放：Kafka。
任务队列、复杂路由、传统消息确认：RabbitMQ。
```

### 这一轮考察点

- 是否理解 Kafka 的事件日志模型。
- 是否知道 RabbitMQ 的队列和路由模型。
- 是否能根据场景做技术选型。

---

## 第三轮：topic、partition、offset

### 面试官提问

解释一下 topic、partition 和 offset 的关系。

### 你的回答

topic 是主题，partition 是分区，offset 是消息的位置。

### 面试官追问

这个太短了。offset 是全局递增的吗？Kafka 能保证 topic 全局有序吗？

### 更好的回答

topic 是 Kafka 中一类消息的逻辑名称，例如 `order.created`。

一个 topic 可以拆成多个 partition。partition 是实际存储和并行处理的单位，每个 partition 都是一个有序追加的日志。

offset 是消息在某个 partition 内的位置。它只在 partition 内有意义，不是整个 topic 的全局编号。

所以：

```text
topic 包含多个 partition。
每个 partition 内部按 offset 顺序追加消息。
不同 partition 的 offset 互不相关。
Kafka 只能保证单个 partition 内有序，不能保证整个 topic 全局有序。
```

如果我希望同一个订单的事件有序，就会用 `order_id` 作为 message key，让同一个订单的事件进入同一个 partition。

但这只能保证订单维度的局部顺序，不代表整个 `order.created` topic 全局有序。

### 这一轮考察点

- 是否知道 offset 不是全局编号。
- 是否理解 partition 内有序。
- 是否能结合 message key 讲顺序性。

---

## 第四轮：message key

### 面试官提问

Kafka 的 message key 有什么用？

### 你的回答

key 可以决定消息发到哪个 partition。

### 面试官追问

如果订单事件用 `user_id` 做 key，会有什么问题？如果不设置 key 呢？

### 更好的回答

message key 主要有三个作用：

1. 决定消息进入哪个 partition。
2. 让同一个 key 的消息进入同一个 partition，从而保证这个 key 维度的局部顺序。
3. 在 log compaction topic 中作为合并依据。

订单事件里，我通常优先考虑用 `order_id` 作为 key。这样同一个订单的 `created`、`paid`、`cancelled` 等事件可以进入同一个 partition，方便保证订单维度的顺序。

如果用 `user_id` 作为 key，可能会让大用户或高频用户形成热点，导致某个 partition 压力特别高。另外，用户维度有序不一定是订单业务真正需要的顺序。

如果不设置 key，消息一般会被客户端分散到多个 partition，吞吐和分布可能更好，但无法保证某个业务维度的顺序。

所以 key 设计不是随便选字段，而是要回答：

```text
业务需要哪个维度有序？
这个字段会不会导致热点？
下游排查时是否方便按 key 追踪？
```

### 这一轮考察点

- 是否理解 key 与 partition 的关系。
- 是否知道热点问题。
- 是否能从业务顺序角度设计 key。

---

## 第五轮：Producer 可靠性

### 面试官提问

Kafka Producer 如何保证消息可靠发送？

### 你的回答

设置 `acks=all`，然后开启重试。

### 面试官追问

只设置 `acks=all` 就够了吗？如果数据库写成功了，但是消息没发出去怎么办？

### 更好的回答

`acks=all` 和 retries 只是 Producer 到 Broker 这段链路的可靠性配置，不等于端到端可靠。

Producer 侧我会考虑：

```text
acks=all。
合理配置 retries 和 delivery timeout。
开启幂等 producer。
消息带 event_id。
发送成功后记录 topic、partition、offset。
发送失败要分类处理。
```

但如果业务里有数据库事务，真正麻烦的是数据库和 Kafka 之间的一致性。

例如订单服务：

```text
订单写库成功。
服务宕机。
Kafka 消息没有发出去。
```

这个问题不是 `acks=all` 能解决的。

我会使用 Outbox Pattern：

```text
在同一个数据库事务里写 orders 表和 outbox_events 表。
事务提交后，后台 publisher 扫描 outbox。
publisher 发送 Kafka。
发送成功后标记 published。
发送失败保留 pending 或 retry 状态。
```

这样只要订单事务提交成功，待发送事件也一定落库。即使 Kafka 短暂不可用，也可以等恢复后继续发送。

### 这一轮考察点

- 是否知道 `acks=all` 的边界。
- 是否能讲出 Outbox。
- 是否理解端到端可靠性。

---

## 第六轮：幂等 Producer

### 面试官提问

什么是 Kafka 的幂等 Producer？

### 你的回答

幂等 Producer 可以防止消息重复。

### 面试官追问

它能防止 Consumer 重复消费吗？能保证业务幂等吗？

### 更好的回答

幂等 Producer 主要解决的是 Producer 因为重试导致同一条消息被重复写入 Broker 的问题。

开启后，Producer 和 Broker 会通过 producer id、序列号等机制识别重复发送，避免同一个 producer 对同一个 partition 的重复追加。

但它的边界要说清楚：

```text
幂等 Producer 解决的是 Producer -> Broker 这段的重复写入问题。
它不等于 Consumer 幂等。
也不能替代业务幂等。
```

Consumer 仍然可能因为处理成功但 offset 提交失败、rebalance、重启、DLQ 重放等原因重复消费。

所以在业务上，我仍然会使用 `event_id`、唯一约束、processed_events 表或状态机来保证 Consumer 幂等。

一句话总结：

```text
幂等 Producer 减少 Kafka 写入重复，业务幂等负责处理消费重复。
```

### 这一轮考察点

- 是否知道幂等 Producer 的边界。
- 是否能区分 Producer 幂等和业务幂等。

---

## 第七轮：Consumer Group

### 面试官提问

Consumer Group 是什么？

### 你的回答

Consumer Group 是一组消费者一起消费消息。

### 面试官追问

如果 topic 有 3 个 partition，同一个 group 启动 10 个 consumer，会怎么样？

### 更好的回答

Consumer Group 是 Kafka 用来实现消费扩展和消费进度隔离的机制。

同一个 group 内，Kafka 会把 topic 的 partition 分配给组内 consumer。通常一个 partition 在同一时刻只会被同一个 group 里的一个 consumer 消费。

如果 topic 有 3 个 partition，同一个 group 启动 10 个 consumer，最多只有 3 个 consumer 能分配到 partition，剩下 7 个会空闲。

所以 Consumer 扩容不是无限加实例。扩容前要先看：

```text
topic partition 数。
consumer 实例数。
每个 partition 的 lag。
单个 handler 的处理能力。
```

不同 consumer group 之间互不影响，可以各自消费同一个 topic。例如库存服务和通知服务可以使用不同 group 消费 `order.created`。

### 这一轮考察点

- 是否知道 group 内 partition 独占。
- 是否知道扩容受 partition 数限制。
- 是否知道不同 group offset 独立。

---

## 第八轮：offset 提交

### 面试官提问

Consumer 的 offset 什么时候提交比较合适？

### 你的回答

消费完就提交。

### 面试官追问

什么叫消费完？如果先提交 offset，再写数据库失败会怎样？

### 更好的回答

对于有业务副作用的消息，我会在业务处理成功后再提交 offset。

完整流程是：

```text
拉取消息。
反序列化和校验。
执行业务处理，例如写数据库。
业务处理成功。
提交 offset。
```

如果先提交 offset，再写数据库，可能出现：

```text
offset 已提交。
数据库写入失败。
consumer 重启后不会再读这条消息。
业务结果丢失。
```

如果业务处理成功，但提交 offset 失败，则可能重复消费。这时依赖业务幂等处理即可。

所以我更能接受重复消费，也不希望业务成功前提前提交 offset。

失败时的策略：

```text
可重试错误：不提交 offset，进入 retry。
不可恢复错误：写 DLQ 成功后提交 offset。
业务成功：提交 offset。
```

### 这一轮考察点

- 是否理解 offset 提交和业务成功的关系。
- 是否知道提前提交的风险。
- 是否接受重复消费并用幂等解决。

---

## 第九轮：重复消费

### 面试官提问

Kafka 为什么会重复消费？

### 你的回答

因为 Kafka 是至少一次消费，所以会重复。

### 面试官追问

能不能说几个具体场景？你怎么解决？

### 更好的回答

重复消费常见原因有：

```text
consumer 处理业务成功，但提交 offset 前宕机。
offset 提交请求失败。
consumer 处理时间太长，引发 rebalance。
rebalance 后未提交的消息被其他 consumer 接管。
人工重放 DLQ。
producer 业务层重复发送了同一个事件。
```

解决方式不是假设它不会重复，而是做业务幂等。

例如库存服务消费订单事件时，我会让事件带 `event_id`，并在数据库里建 processed_events 表：

```sql
CREATE TABLE processed_events (
    event_id text PRIMARY KEY,
    processed_at timestamptz NOT NULL DEFAULT now()
);
```

处理时：

```text
开启数据库事务。
先插入 event_id。
如果唯一约束冲突，说明处理过，直接返回成功。
执行库存预占。
提交事务。
提交 offset。
```

同时对库存预占表按 `order_id` 做唯一约束，防止同一订单重复预占。

### 这一轮考察点

- 是否能列举重复消费场景。
- 是否能给出数据库级幂等方案。
- 是否知道唯一约束比内存去重可靠。

---

## 第十轮：Consumer Lag

### 面试官提问

Consumer Lag 是什么？线上 lag 很高你怎么排查？

### 你的回答

Lag 就是消息堆积。Lag 高就加 consumer。

### 面试官追问

加 consumer 一定有用吗？如果只有一个 partition lag 高呢？

### 更好的回答

Consumer Lag 表示 consumer group 的已提交 offset 落后于 partition 最新 offset 的数量。

简单说：

```text
lag = partition 最新 offset - consumer group 已提交 offset
```

Lag 高只是现象，不是原因。我会按顺序排查：

```text
1. 确认 topic、group、partition。
2. 看是所有 partition lag 高，还是单个 partition lag 高。
3. 看 consumer 实例是否正常。
4. 看 consumer 错误日志和失败率。
5. 看 handler 耗时是否升高。
6. 看下游数据库、Redis、HTTP 服务是否变慢。
7. 看 producer 写入速率是否突然上涨。
8. 看是否发生 rebalance。
9. 看 key 是否倾斜导致热点 partition。
```

如果所有 partition lag 都高，可能是整体处理能力不足或下游依赖慢。

如果只有一个 partition lag 高，可能是 key 热点、某类消息处理慢，或者该 partition 对应 consumer 异常。此时单纯增加 consumer 不一定有用，因为同一个 group 内一个 partition 同时只能由一个 consumer 消费。

### 这一轮考察点

- 是否知道 lag 的定义。
- 是否知道加 consumer 受 partition 限制。
- 是否能区分全局堆积和单 partition 热点。

---

## 第十一轮：Rebalance

### 面试官提问

什么是 Rebalance？它有什么影响？

### 你的回答

Rebalance 是重新分配 partition，可能会影响消费。

### 面试官追问

什么情况下会触发？Go 服务怎么降低影响？

### 更好的回答

Rebalance 是 consumer group 重新分配 partition 的过程。

触发场景包括：

```text
新增 consumer。
consumer 下线。
consumer 心跳超时。
topic partition 数变化。
consumer 处理太久，超过 max poll interval。
```

影响包括：

- 消费短暂停顿。
- partition 重新分配。
- 未提交 offset 的消息可能被重复处理。
- lag 可能短暂上升。

Go 服务降低影响的方式：

```text
合理设置 session timeout、heartbeat interval、max poll interval。
handler 不要长时间阻塞 poll。
滚动发布时控制节奏。
收到 SIGTERM 后优雅退出。
停止拉取新消息。
等待正在处理的消息完成。
成功处理后提交 offset。
再关闭 consumer。
```

同时要监控 rebalance 次数和持续时间。如果发布期间 rebalance 频繁，需要检查退出流程和参数设置。

### 这一轮考察点

- 是否知道 rebalance 触发条件。
- 是否知道 rebalance 和重复消费的关系。
- 是否能讲 Go 服务优雅退出。

---

## 第十二轮：DLQ

### 面试官提问

什么是 DLQ？什么时候用？

### 你的回答

DLQ 是死信队列，消费失败就放进去。

### 面试官追问

是不是所有失败都应该进 DLQ？库存不足要不要进 DLQ？

### 更好的回答

DLQ 是 Dead Letter Queue，用来存放无法正常处理的消息，避免坏消息一直阻塞主消费链路。

但不是所有失败都应该进 DLQ。

我会先区分：

```text
技术失败：JSON 格式错误、schema 不兼容、字段缺失、多次重试仍失败。
临时失败：数据库超时、下游限流、网络抖动。
业务失败：库存不足、订单已取消、优惠券不可用。
```

技术失败可以进入 DLQ。

临时失败通常先进入 retry，多次失败后再进 DLQ。

业务失败不一定是异常，例如库存不足应该建模为 `inventory.reserve_failed` 事件，通知订单服务取消订单或进入待处理状态，而不是简单丢进 DLQ。

DLQ 里应该保留：

```text
原 topic、partition、offset、key、value。
event_id。
错误原因。
失败时间。
重试次数。
trace_id。
```

并且 DLQ 必须有告警、查询和受控重放工具。DLQ 不是垃圾桶。

### 这一轮考察点

- 是否能区分技术失败和业务失败。
- 是否知道 DLQ 需要运维闭环。
- 是否知道 DLQ 重放也要求幂等。

---

## 第十三轮：Retry 设计

### 面试官提问

Kafka 消费失败后如何设计重试？

### 你的回答

失败了就重试几次，不行放 DLQ。

### 面试官追问

怎么避免立即重试打爆下游？重试消息怎么记录次数？

### 更好的回答

我会先对错误分类：

```text
临时错误：数据库超时、下游 HTTP 429、网络抖动。
不可恢复错误：JSON 格式错误、缺少必要字段、schema 不兼容。
业务结果：库存不足、订单状态不允许。
```

临时错误可以重试，但不能无限立即重试，否则可能在下游已经故障时继续放大压力。

常见设计是 retry topic：

```text
main topic
-> retry 1min
-> retry 5min
-> retry 30min
-> DLQ
```

消息里或 header 中记录：

```text
event_id
retry_count
last_error
first_failed_at
last_failed_at
```

达到最大重试次数后进入 DLQ。

重试消息仍然可能重复投递，所以 consumer 必须幂等。

### 这一轮考察点

- 是否有错误分类。
- 是否知道退避重试。
- 是否知道 retry 和 DLQ 的边界。

---

## 第十四轮：Outbox

### 面试官提问

你刚才提到 Outbox，能详细说说吗？

### 你的回答

Outbox 就是把消息先存在数据库，然后再发 Kafka。

### 面试官追问

为什么这能解决一致性问题？Outbox 会不会重复发送？

### 更好的回答

Outbox Pattern 解决的是业务数据库事务和 Kafka 消息发送之间的一致性缺口。

不用 Outbox 时有两个典型风险：

```text
先写库后发消息：数据库成功，Kafka 发送失败，事件丢失。
先发消息后写库：Kafka 成功，数据库回滚，下游消费到不存在的事实。
```

Outbox 做法是：

```text
开启数据库事务。
写业务表，例如 orders。
写 outbox_events 表，记录要发布的事件。
提交事务。
后台 publisher 扫描 pending outbox。
发送 Kafka。
发送成功后标记 published。
发送失败记录 error 并等待重试。
```

它的价值是：业务数据和待发送事件在同一个本地事务里提交。只要订单创建成功，事件就一定在 outbox 表中。

Outbox 可能重复发送，例如发送 Kafka 成功但更新 outbox 状态失败。所以 consumer 仍然必须通过 event_id 做幂等。

所以 Outbox 解决发布可靠性，Consumer 幂等解决重复投递副作用。

### 这一轮考察点

- 是否知道 Outbox 解决什么。
- 是否知道 Outbox 也可能重复发送。
- 是否能说清和 consumer 幂等的关系。

---

## 第十五轮：Exactly Once

### 面试官提问

Kafka 不是支持 Exactly Once 吗？那为什么还要做幂等？

### 你的回答

Exactly Once 可以保证消息只消费一次，但是实际项目还要幂等。

### 面试官追问

这句话有点矛盾。你具体解释一下 Exactly Once 的边界。

### 更好的回答

Kafka 的 Exactly Once 有适用边界，主要用于 Kafka 内部的事务性生产、消费位移提交、以及 Kafka Streams 这类 consume-process-produce 场景。

但在普通 Go 后端项目中，consumer 通常会操作外部系统，例如 PostgreSQL、Redis、HTTP 下游。Kafka 的 Exactly Once 不能自动保证外部数据库只被更新一次。

例如：

```text
consumer 写 PostgreSQL 成功。
提交 offset 前进程崩溃。
消息再次被消费。
```

这时 Kafka 事务不能自动回滚或识别 PostgreSQL 里的业务副作用。

所以我的理解是：

```text
Kafka Exactly Once 可以解决 Kafka 内部特定链路的一致性问题。
涉及外部数据库和业务副作用时，仍然需要业务幂等。
```

在 Go 业务系统里，我更常用的是 at-least-once + 幂等处理 + Outbox。

### 这一轮考察点

- 是否知道 Exactly Once 的边界。
- 是否能把 Kafka 内部语义和外部数据库区分开。

---

## 第十六轮：消息顺序

### 面试官提问

Kafka 如何保证消息顺序？

### 你的回答

Kafka 可以保证顺序。

### 面试官追问

保证哪里的顺序？如果一个 topic 有多个 partition 呢？

### 更好的回答

Kafka 只能保证单个 partition 内的消息顺序，不保证整个 topic 的全局顺序。

如果 topic 有多个 partition，不同 partition 之间的消息没有全局先后关系。

如果业务需要某个维度有序，例如同一个订单的状态变更有序，我会：

```text
使用 order_id 作为 message key。
让同一订单事件进入同一个 partition。
consumer 按 partition 顺序处理。
避免同一 partition 内乱序并发处理。
业务层使用状态机或版本号防止状态倒退。
```

也要注意 Producer 重试和并发发送可能影响顺序，所以重要业务会开启幂等 producer，并控制相关参数。

最终我会把顺序性理解为：

```text
Kafka 提供 partition 内顺序，业务通过 key 设计和状态机维护业务顺序。
```

### 这一轮考察点

- 是否知道 partition 内有序。
- 是否知道业务顺序要靠 key 和状态机共同保证。

---

## 第十七轮：消息丢失排查

### 面试官提问

线上有人说 Kafka 消息丢了，你怎么排查？

### 你的回答

我会看日志，看 producer 有没有发成功，consumer 有没有消费。

### 面试官追问

能不能更系统一点？从哪几个阶段排查？

### 更好的回答

我会先澄清所谓“丢了”是哪个阶段不见了，然后按 event_id 追踪。

排查链路：

```text
1. 业务是否真的生成了事件。
2. 如果使用 Outbox，outbox 表是否有记录。
3. publisher 是否扫描到 outbox。
4. producer 发送是否成功，是否拿到 topic、partition、offset。
5. Kafka topic 中是否存在对应 offset。
6. consumer group 是否消费过。
7. consumer 是否提前提交 offset。
8. handler 是否处理失败。
9. 消息是否进入 retry 或 DLQ。
10. retention 是否已清理。
11. 是否查错了 topic、group 或环境。
```

很多“消息丢失”其实不是 Kafka 把消息弄丢，而是：

- producer 根本没发成功。
- 数据库成功但事件没发布。
- consumer 提前提交 offset。
- 消息进了 DLQ 没人看。
- retention 到期。
- 查错了 consumer group。

所以日志里必须有 event_id、topic、partition、offset、group 和 trace_id。

### 这一轮考察点

- 是否能端到端排查。
- 是否知道丢失可能发生在业务、producer、broker、consumer、DLQ 多个阶段。

---

## 第十八轮：Kafka 不可用

### 面试官提问

如果 Kafka 集群不可用，你的业务系统怎么办？

### 你的回答

Kafka 不可用的话，消息发送失败，等恢复后重试。

### 面试官追问

用户下单接口还能成功吗？你怎么避免请求被 Kafka 拖垮？

### 更好的回答

这取决于业务链路如何设计。

如果订单接口同步依赖 Kafka 发送成功，那么 Kafka 不可用会导致下单接口失败或变慢。这种设计对关键业务不太友好。

我更倾向使用 Outbox：

```text
用户下单时，只写订单表和 outbox 表。
这两个操作在同一个数据库事务里完成。
Kafka 是否可用不直接阻塞用户请求。
后台 publisher 负责把 outbox 事件发送到 Kafka。
Kafka 不可用时，outbox 保持 pending。
Kafka 恢复后继续发送。
```

同时要监控：

```text
outbox pending count。
outbox pending age。
publisher send error rate。
Kafka broker 可用性。
```

如果 pending age 超过阈值，要告警，因为下游库存、通知等系统会延迟处理。

所以 Kafka 不可用时，核心业务是否还能成功，要看是否把 Kafka 发送从用户请求链路中解耦出来。

### 这一轮考察点

- 是否知道同步发送 Kafka 会影响主链路。
- 是否能用 Outbox 降低 Kafka 故障影响。
- 是否知道 outbox 积压也要监控。

---

## 第十九轮：Go 项目封装

### 面试官提问

在 Go 项目里你怎么封装 Kafka Producer 和 Consumer？

### 你的回答

我会写一个 producer 文件和 consumer 文件，调用 Kafka 客户端。

### 面试官追问

业务代码会直接依赖 Kafka 客户端吗？怎么测试？

### 更好的回答

我会把 Kafka 当成基础设施适配层，不让业务 service 直接依赖具体 Kafka 客户端。

Producer 侧：

```go
type OrderEventPublisher interface {
    PublishOrderCreated(ctx context.Context, event OrderCreatedEvent) error
}
```

业务 service 依赖这个接口。Kafka adapter 实现这个接口，内部负责：

```text
序列化事件。
设置 topic。
设置 key。
设置 headers。
调用 Kafka 客户端。
记录 topic、partition、offset。
```

Consumer 侧：

```go
type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

外层 consumer framework 负责：

```text
拉取消息。
反序列化。
调用 handler。
根据 error 类型决定 commit、retry 或 DLQ。
处理优雅退出。
```

测试时可以用 fake publisher 或 fake handler，不需要所有单元测试都启动 Kafka。真实 Kafka 只放在集成测试里。

### 这一轮考察点

- 是否有工程分层意识。
- 是否知道业务代码不应强依赖 Kafka 客户端。
- 是否能说明测试策略。

---

## 第二十轮：Kafka 上线评审

### 面试官提问

如果让你从 0 到 1 上线一条 Kafka 链路，你会评审哪些内容？

### 你的回答

我会检查 topic、producer、consumer 和监控。

### 面试官追问

能不能列一个更完整的上线 checklist？

### 更好的回答

我会从业务、可靠性、运维和回滚几个方面评审。

Checklist：

```text
1. 业务事件是否定义清楚。
2. topic 名称、partition 数、replication factor 是否合理。
3. message key 是否符合业务顺序要求。
4. schema 是否包含 event_id、schema_version、occurred_at、trace_id。
5. producer 是否有可靠性参数和发送失败处理。
6. 是否需要 Outbox。
7. consumer 是否手动提交 offset。
8. consumer 是否幂等。
9. retry 和 DLQ 是否设计。
10. DLQ 是否有告警和重放工具。
11. ACL 权限是否最小化。
12. Prometheus 指标和日志字段是否齐全。
13. lag、失败率、DLQ、outbox pending age 是否有告警。
14. 是否做过压测。
15. 是否有灰度和回滚方案。
```

我不会只看代码能不能跑通，还要确认这条链路上线后能观察、能恢复、能重放、能回滚。

### 这一轮考察点

- 是否有生产意识。
- 是否知道 Kafka 链路不是只有 producer 和 consumer。
- 是否能说出上线、观测、回滚。

---

## 第二十一轮：综合项目追问

### 面试官提问

你能完整讲一下订单创建后，Kafka 消息从产生到消费的生命周期吗？

### 你的回答

订单服务创建订单，然后发 Kafka，库存服务消费，处理成功后提交 offset。

### 面试官追问

如果中间任意一步失败呢？你怎么保证最终能恢复？

### 更好的回答

完整链路我会这样讲：

```text
1. 用户调用创建订单接口。
2. 订单服务开启数据库事务。
3. 写 orders 表。
4. 写 outbox_events 表，事件类型为 order.created。
5. 提交事务。
6. 后台 publisher 扫描 pending outbox。
7. publisher 按 order_id 作为 key 发送 Kafka。
8. 发送成功后记录 topic、partition、offset，并标记 outbox 为 published。
9. 库存服务 consumer group 拉取 order.created。
10. consumer 反序列化并校验 schema_version。
11. 在数据库事务中插入 processed_events。
12. 如果 event_id 已存在，说明重复消费，直接返回成功。
13. 执行库存预占。
14. 提交业务事务。
15. 提交 Kafka offset。
16. 指标记录 handler duration、success count 和 lag。
```

失败恢复：

```text
订单事务失败：orders 和 outbox 都不会写入。
事务成功但 Kafka 不可用：outbox 保持 pending，恢复后继续发。
Kafka 发送成功但 outbox 状态更新失败：可能重复发送，consumer 幂等处理。
consumer 处理成功但 offset 提交失败：下次重复消费，processed_events 幂等跳过。
consumer 处理临时失败：进入 retry。
消息格式错误：进入 DLQ 并告警。
DLQ 修复后：通过受控工具限速重放。
```

这条链路的核心是：Outbox 保证事件不会因为进程故障丢失，consumer 幂等保证重复投递不会产生重复副作用，retry 和 DLQ 保证失败可恢复，可观测性保证问题能被发现。

### 这一轮考察点

- 是否能串起完整生命周期。
- 是否能逐步处理失败场景。
- 是否真正理解可靠事件驱动。

---

## 第二十二轮：面试收尾问题

### 面试官提问

你觉得自己对 Kafka 还有哪些不足？

### 你的回答

我觉得还需要继续深入学习 Kafka 底层原理。

### 面试官追问

具体是哪些方面？你后面怎么补？

### 更好的回答

我目前更熟悉 Kafka 在 Go 后端业务系统里的使用，包括 topic 设计、producer、consumer group、offset、幂等、Outbox、DLQ 和 lag 排查。

我觉得还可以继续深入的方向有：

```text
Kafka broker 内部更多实现细节。
KRaft 元数据管理。
更大规模集群下的容量规划。
Kafka Streams 和事务语义。
跨机房容灾和 MirrorMaker。
更系统的性能压测。
```

但对后端开发岗位来说，我现在重点是把 Kafka 用在可靠的业务链路中，能处理重复消费、消息失败、异步一致性和生产排障。后续我会通过做一个完整的订单事件驱动项目和压测实验继续补齐这些能力。

这个回答比单纯说“我都懂”更可信，也能体现学习边界。

### 这一轮考察点

- 是否能客观看待自己的能力。
- 是否知道后续学习方向。
- 是否能把不足说成成长计划。

---

## 复盘：面试回答的升级公式

一个普通回答通常是：

```text
Kafka 可以做消息队列。
```

一个更好的回答应该是：

```text
Kafka 在我的项目里用于订单事件驱动。
它解决的是订单服务和多个下游系统之间的异步解耦。
我用 order_id 做 key 保证订单维度顺序。
用 Outbox 解决数据库事务和消息发送一致性。
consumer 使用 event_id 和唯一约束保证幂等。
失败时按错误类型进入 retry 或 DLQ。
通过 lag、DLQ、发送失败率和 outbox pending age 做观测。
```

也就是说，每次回答都尽量包含：

```text
概念是什么。
为什么需要。
项目里怎么用。
有什么风险。
如何兜底。
如何观测。
```

这样你的 Kafka 面试表达就会从“背概念”变成“讲工程设计”。
