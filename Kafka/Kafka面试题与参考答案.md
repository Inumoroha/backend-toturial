# Kafka 面试题与参考答案

这份文档用于准备 Go 后端工程师面试中常见的 Kafka 问题。

它不是让你背标准答案，而是帮助你做到：

```text
听懂面试官真正想考什么。
能用自己的项目经验解释 Kafka。
能把 topic、partition、offset、consumer group、幂等、DLQ、Outbox 讲成一条完整链路。
遇到追问时知道往可靠性、性能、可观测性和生产实践方向展开。
```

建议复习方式：

1. 先看题目，自己口头回答一遍。
2. 再看参考答案，补齐遗漏点。
3. 最后把答案改成适合自己的表达。
4. 尽量结合订单系统、库存系统、消息通知、电商事件驱动项目来讲。

面试时不要只背名词。Kafka 面试最重要的是把这些问题讲成工程判断：

```text
为什么要用 Kafka？
消息怎么分区？
如何保证不乱序？
如何面对重复消费？
消费失败怎么办？
如何观察 lag？
Go 服务如何优雅退出？
生产环境如何上线和回滚？
```

---

## 一、Kafka 基础认知类

### 1. Kafka 是什么？

参考回答：

Kafka 是一个分布式事件流平台，也可以理解为高吞吐、可持久化、可水平扩展的消息系统。

在后端系统中，Kafka 常用于：

- 异步解耦。
- 削峰填谷。
- 事件驱动架构。
- 日志和行为数据采集。
- 多个下游系统订阅同一份业务事件。

例如订单服务创建订单后，可以把 `order.created` 事件写入 Kafka，库存服务、积分服务、通知服务分别消费这个事件。订单服务不用同步调用所有下游，系统耦合会降低。

可以这样总结：

```text
Kafka 把业务中已经发生的事实，以可持久化、可订阅、可扩展的方式传递给其他系统。
```

追问点：

- Kafka 和普通消息队列有什么区别？
- Kafka 为什么适合高吞吐？
- Kafka 是消息队列还是事件流平台？

---

### 2. 为什么后端系统要引入 Kafka？

参考回答：

引入 Kafka 通常不是因为“技术先进”，而是因为系统中出现了同步调用难以处理的问题。

典型原因包括：

- 上游不想被下游接口耗时拖慢。
- 一个业务事件需要被多个系统处理。
- 瞬时流量很高，需要缓冲。
- 下游系统可能短暂不可用，需要消息暂存。
- 需要保留事件历史，便于重放和审计。

例如电商下单场景：

```text
订单服务负责创建订单。
库存服务消费订单事件并预占库存。
通知服务消费订单事件并发送短信。
数据分析服务消费订单事件并生成报表。
```

如果全部同步调用，订单接口会依赖多个下游系统的稳定性。引入 Kafka 后，订单服务只需要发布订单事件，下游系统异步处理。

但也要补充：

Kafka 会引入最终一致性、重复消费、消息堆积、运维复杂度等问题。不是所有场景都适合用 Kafka。

---

### 3. 什么场景不适合使用 Kafka？

参考回答：

Kafka 不适合所有后端交互。

不太适合的场景有：

- 用户请求必须立即拿到强一致结果。
- 调用链很短，同步接口更简单。
- 业务量很小，引入 Kafka 反而增加复杂度。
- 团队还没有处理重复消费、失败重试、DLQ 和监控告警的能力。
- 消息语义不稳定，事件结构频繁变化。

例如用户点击支付按钮后，页面通常需要尽快知道支付是否成功，这类强反馈场景不能简单丢给 Kafka 后就立即返回成功。

可以这样回答：

```text
Kafka 更适合传递“已经发生的业务事实”，不适合替代所有同步接口调用。
```

---

### 4. Kafka 和 RabbitMQ 有什么区别？

参考回答：

Kafka 和 RabbitMQ 都能做消息通信，但设计重点不同。

Kafka 更像分布式事件日志：

- 消息按 partition 追加写入。
- 消息会按保留策略保存一段时间。
- 多个 consumer group 可以重复消费同一份数据。
- 吞吐高，适合事件流、日志、数据管道、大规模异步处理。

RabbitMQ 更偏传统消息队列：

- 强调队列、交换机、路由规则。
- 消息通常被消费后确认并删除。
- 路由能力灵活。
- 适合任务分发、复杂路由、低延迟消息投递。

面试中可以说：

```text
如果是订单事件流、日志采集、多个系统订阅同一事件，我更倾向 Kafka。
如果是复杂路由、任务队列、请求响应式消息，我会考虑 RabbitMQ。
```

不要简单说“Kafka 比 RabbitMQ 好”。更好的表达是：两者适合的场景不同。

---

### 5. Kafka 为什么吞吐高？

参考回答：

Kafka 吞吐高，主要来自几个设计：

- 顺序追加写磁盘，避免大量随机写。
- partition 可以水平扩展，多个 broker 并行读写。
- producer 支持 batch 批量发送。
- consumer 支持批量拉取。
- 利用操作系统 page cache。
- 网络和磁盘操作都尽量批量化。
- 消息格式紧凑，适合顺序传输。

可以这样讲：

```text
Kafka 不是不写磁盘，而是用顺序写、批量写和 page cache 把磁盘写入变得很高效。
```

追问点：

- 顺序写为什么快？
- batch 和 linger 如何影响吞吐？
- 压缩为什么能提升吞吐？

---

### 6. Kafka 是推模型还是拉模型？

参考回答：

Kafka 的 consumer 是拉模型。

也就是说，consumer 主动向 broker 拉取消息，而不是 broker 主动推送给 consumer。

拉模型的好处是：

- consumer 可以按自己的处理能力拉取。
- 不容易被 broker 推爆。
- consumer 可以控制批量大小。
- 不同 consumer group 可以按自己的节奏消费。

缺点是：

- consumer 需要处理轮询、等待和 lag。
- 如果参数设置不合理，可能出现拉取效率低或延迟高。

在 Go 后端中，拉取到消息后，通常要：

```text
反序列化 -> 校验 -> 执行业务处理 -> 成功后提交 offset -> 失败后 retry 或 DLQ
```

---

### 7. Kafka 中的消息会不会消费后就删除？

参考回答：

不会。

Kafka 的消息是否删除，不取决于某个 consumer 是否消费过，而取决于 topic 的保留策略，例如：

- 保留多长时间。
- 保留多少磁盘空间。
- 是否启用 log compaction。

这和很多传统消息队列不同。Kafka 中多个 consumer group 可以各自消费同一份消息，它们的消费进度互不影响。

例如：

```text
inventory-service 消费了 order.created。
notification-service 仍然可以消费同一条 order.created。
新建一个 analysis-service group，也可以从 earliest 开始重新读历史消息。
```

---

### 8. Kafka 中的事件和消息有什么区别？

参考回答：

消息是技术层面的说法，表示 Kafka 中的一条 record。

事件更偏业务语义，表示某件事情已经发生。

例如：

```text
消息：Kafka record，包含 key、value、headers、timestamp、topic、partition、offset。
事件：order.created，表示订单已经创建。
```

在事件驱动架构中，更推荐把 Kafka 中的业务消息理解为事件，因为这能提醒我们：

- 事件应该表达事实，而不是命令。
- 事件名称应该清晰，例如 `order.created`。
- 事件应该有版本、event_id、occurred_at 等字段。
- 下游消费事件时要接受最终一致性。

---

### 9. Kafka 中的一条消息通常包含哪些信息？

参考回答：

一条 Kafka record 通常包含：

- topic：消息所在的主题。
- partition：消息所在分区。
- offset：消息在分区内的位置。
- key：用于分区选择，也常用于保证局部顺序。
- value：消息内容。
- headers：元数据，例如 trace_id、event_id、schema_version。
- timestamp：消息时间戳。

Go 后端日志中建议记录：

```text
topic
partition
offset
key
event_id
trace_id
consumer_group
error
```

这些字段对排查重复消费、消息丢失、顺序问题和 lag 都很重要。

---

### 10. Kafka 和数据库 binlog 有什么区别？

参考回答：

数据库 binlog 是数据库变更日志，记录数据变更。Kafka 是通用事件流平台，可以承载业务事件、日志、指标、数据同步事件等。

区别可以这样理解：

- binlog 记录的是数据库层面的变化。
- Kafka 事件记录的是业务层面的事实。
- binlog 更适合 CDC 数据同步。
- Kafka 业务事件更适合系统间解耦和业务流程编排。

例如订单表插入一行，binlog 能记录这次 insert。但业务事件 `order.created` 可以包含更清晰的业务语义，例如订单来源、用户信息摘要、优惠信息、事件版本等。

---

## 二、核心概念与存储模型类

### 11. 什么是 topic？

参考回答：

topic 是 Kafka 中消息的逻辑分类，可以理解为一类事件流的名字。

例如：

```text
order.created
order.paid
inventory.reserved
payment.succeeded
```

producer 向 topic 写消息，consumer 从 topic 读消息。

topic 的设计要贴近业务语义，不建议设计成过于宽泛的名字，例如 `all_events`。太宽泛会导致 schema 混乱、权限难控、排查困难。

好的 topic 名称通常具备：

- 业务域清楚。
- 事件语义清楚。
- 粒度适中。
- 便于权限和监控管理。

---

### 12. 什么是 partition？

参考回答：

partition 是 topic 的物理分片。一个 topic 可以拆成多个 partition，每个 partition 是一个有序追加的日志。

partition 的作用：

- 提升并行写入能力。
- 提升并行消费能力。
- 支持数据分布到多个 broker。
- 在同一个 partition 内保证消息顺序。

需要注意：

```text
Kafka 只能保证同一个 partition 内的消息有序，不能保证整个 topic 全局有序。
```

所以如果希望同一个订单的事件有序，通常会用 `order_id` 作为 message key，让同一个订单进入同一个 partition。

---

### 13. 什么是 offset？

参考回答：

offset 是消息在某个 partition 内的递增位置。

关键点：

- offset 只在 partition 内有意义。
- 不同 partition 的 offset 互不相关。
- offset 不是 topic 的全局编号。
- consumer group 会记录自己在每个 partition 上消费到哪里。

例如：

```text
topic=order.created
partition=0 offset=100
partition=1 offset=100
```

这两条消息不是同一条，它们只是分别位于不同 partition 的第 100 个位置。

---

### 14. 什么是 broker？

参考回答：

broker 是 Kafka 集群中的一个服务节点，负责存储和处理 topic partition 的读写请求。

一个 Kafka 集群通常由多个 broker 组成。topic 的 partition 会分布在不同 broker 上，从而实现水平扩展。

broker 的职责包括：

- 接收 producer 写入。
- 响应 consumer 拉取。
- 存储 partition 日志。
- 复制副本数据。
- 参与 leader 选举和集群元数据管理。

面试时可以补一句：

```text
Kafka 的扩展能力很大程度来自 partition 在多个 broker 上的分布。
```

---

### 15. 什么是 leader 和 follower？

参考回答：

Kafka 中每个 partition 可以有多个副本，其中一个是 leader，其他是 follower。

通常：

- producer 向 leader 写入消息。
- consumer 从 leader 读取消息。
- follower 从 leader 同步数据。
- leader 挂掉后，Kafka 会从合适的 follower 中选出新的 leader。

这样做的目的是提高可用性和容错能力。

但要注意：

```text
副本不是用来提高同一个 partition 的消费并行度的。
消费并行度主要由 partition 数和 consumer group 内消费者数量决定。
```

---

### 16. 什么是 ISR？

参考回答：

ISR 是 In-Sync Replicas 的缩写，表示当前和 leader 保持同步的一组副本。

如果 follower 落后太多，可能会被移出 ISR。等它追上后，再重新加入 ISR。

ISR 和可靠性有关。比如 `acks=all` 时，leader 通常需要等待 ISR 中的副本确认，才认为写入成功。

需要注意：

- ISR 不是所有副本。
- ISR 是当前仍然同步的副本集合。
- ISR 收缩可能表示 broker 故障、网络抖动或磁盘问题。

如果生产环境频繁出现 ISR shrink，需要重点排查集群健康。

---

### 17. replication factor 是什么？

参考回答：

replication factor 表示 partition 的副本数量。

例如 replication factor 为 3，表示每个 partition 有 3 份副本，分布在不同 broker 上。

它的作用是提高容错能力：

- 一个 broker 宕机时，仍然可以从其他副本恢复。
- leader 失败后，可以选举新的 leader。

但副本数越多，也会带来：

- 更多磁盘占用。
- 更多网络复制开销。
- 写入确认成本可能增加。

常见生产配置是 3 副本，但具体要看集群规模、数据重要性和成本。

---

### 18. Kafka 如何保证消息顺序？

参考回答：

Kafka 只保证同一个 partition 内的消息顺序。

如果要保证某个业务维度的顺序，需要让该维度的消息进入同一个 partition。常见做法是设置 message key。

例如：

```text
订单事件使用 order_id 作为 key。
同一订单的 created、paid、cancelled 进入同一个 partition。
consumer 按 partition 顺序消费。
```

但这并不代表整个 topic 有序。

还要补充：

- producer 重试可能影响顺序，需要关注幂等和 in-flight 参数。
- consumer 并发处理同一 partition 内消息时，可能破坏业务顺序。
- 业务层最好用状态机或版本号防止状态倒退。

---

### 19. partition 数是不是越多越好？

参考回答：

不是。

partition 数多的好处：

- 提高并行写入能力。
- 提高 consumer group 并行消费能力。
- 更容易分散数据。

但 partition 数过多也有代价：

- broker 管理更多日志文件。
- leader 选举和元数据管理成本增加。
- rebalance 时间可能变长。
- 小流量 topic 过度分区会浪费资源。

设计 partition 数时要考虑：

```text
峰值写入吞吐。
单个 consumer 的处理能力。
期望 consumer 并行度。
未来增长空间。
broker 资源。
```

---

### 20. 增加 partition 会有什么风险？

参考回答：

增加 partition 可以提高并行度，但也可能带来顺序和分布变化。

风险包括：

- key 到 partition 的映射可能改变。
- 某些业务维度的顺序预期可能被影响。
- consumer group 会发生 rebalance。
- 运维和监控成本增加。

如果业务强依赖同一个 key 的顺序，要特别谨慎。虽然同一个 key 在新分区数量下仍会被映射到某个 partition，但扩分区前后，同一个 key 可能映射到不同 partition，历史消息和新消息可能分布在不同 partition 中。

所以扩 partition 前要评估：

```text
是否依赖跨历史消息的顺序。
是否能接受短暂 rebalance。
是否需要灰度或新 topic 迁移。
```

---

### 21. Kafka 的消息存储在哪里？

参考回答：

Kafka 消息存储在 broker 本地磁盘上，以 partition 为单位组织成日志文件。

每个 partition 对应一组 log segment 文件。消息按顺序追加到日志末尾。

这种设计的好处是：

- 顺序写磁盘效率高。
- segment 便于按时间或大小滚动。
- retention 清理时可以按 segment 删除。
- consumer 可以通过 offset 快速定位消息位置。

Kafka 使用磁盘并不意味着慢。它依赖顺序写、page cache 和批量 IO 获得高吞吐。

---

### 22. 什么是 segment？

参考回答：

segment 是 Kafka partition 日志的分段文件。

一个 partition 的消息不会全部写在一个无限增长的文件中，而是分成多个 segment。

segment 的作用：

- 控制单个文件大小。
- 方便日志滚动。
- 方便过期数据删除。
- 方便索引和查找。

例如 retention 到期时，Kafka 可以删除整个旧 segment，而不是逐条删除消息。

---

### 23. Kafka 的 retention 是什么？

参考回答：

retention 是 Kafka 的消息保留策略。

常见维度：

- 按时间保留，例如保留 3 天。
- 按大小保留，例如最多保留 500GB。

retention 到期后，消息可能被清理。它与 consumer 是否已经消费无关。

例如：

```text
topic 保留 3 天。
某个 consumer group 停止 5 天。
恢复后可能无法消费 5 天前的消息，因为数据已经被清理。
```

所以生产环境要根据业务恢复能力设置 retention。如果下游可能长时间停机，就要增加保留时间或设计补偿机制。

---

### 24. 什么是 log compaction？

参考回答：

log compaction 是 Kafka 的日志压缩策略，核心思想是对同一个 key 尽量保留最新值。

适合场景：

- 用户最新资料。
- 商品最新价格。
- 配置最新状态。
- 账户当前状态快照。

不适合场景：

- 订单创建流水。
- 支付审计流水。
- 需要保留完整历史事件的业务。

要注意：

```text
compaction 不是按时间删除旧消息，而是按 key 合并旧值。
```

如果业务需要完整审计链路，不要轻易使用 compacted topic。

---

### 25. Kafka 的 Controller 或 KRaft 是什么？

参考回答：

Kafka 需要一个集群管理角色来维护元数据，例如 broker 列表、topic、partition、leader 分布等。

早期 Kafka 依赖 ZooKeeper 管理元数据。后来 Kafka 引入 KRaft 模式，用 Kafka 自身的 quorum 管理元数据，逐步替代 ZooKeeper。

面试中不需要展开太多内部实现，可以表达：

```text
Controller 负责集群元数据管理和 partition leader 选举等工作。
KRaft 是 Kafka 新版的元数据管理机制，减少了对 ZooKeeper 的依赖。
```

实际项目中更重要的是知道：broker、topic、partition leader、ISR 这些状态都属于集群元数据范畴。

---

## 三、Producer 生产者类

### 26. Kafka Producer 发送消息的基本流程是什么？

参考回答：

Producer 发送消息大致流程：

```text
构造消息
序列化 key 和 value
根据 topic 和 key 选择 partition
写入本地 buffer
按 batch 和 linger 聚合
发送到对应 partition leader
等待 broker 响应
成功后返回 metadata
失败后根据配置重试或返回错误
```

Go 后端中通常还会封装一层业务接口，例如：

```go
type EventPublisher interface {
    PublishOrderCreated(ctx context.Context, event OrderCreatedEvent) error
}
```

业务层不应该到处感知 Kafka 的底层细节。

---

### 27. message key 有什么作用？

参考回答：

message key 主要有几个作用：

- 决定消息进入哪个 partition。
- 保证同一个 key 的消息进入同一个 partition。
- 支持同一业务维度的局部顺序。
- 在 log compaction topic 中作为压缩依据。

例如订单事件用 `order_id` 作为 key，可以让同一个订单的事件进入同一个 partition，保证该订单维度内的顺序。

但 key 设计要避免热点。例如用 `shop_id` 作为 key，如果某个大商家流量特别高，就可能导致单个 partition 压力过大。

---

### 28. 如果不设置 key 会怎样？

参考回答：

如果不设置 key，producer 通常会使用轮询或粘性分区等策略把消息分布到不同 partition，具体行为取决于客户端实现。

好处：

- 分布可能更均匀。
- 吞吐较好。

缺点：

- 无法保证某个业务维度的顺序。
- 下游排查时不容易按业务 key 追踪。
- compacted topic 中没有 key 会失去语义。

如果业务不关心顺序，例如普通日志采集，可以不设置 key。如果是订单、支付、库存等事件，通常应该认真设计 key。

---

### 29. acks 参数有什么作用？

参考回答：

`acks` 控制 producer 需要等待 broker 确认到什么程度才认为消息发送成功。

常见取值：

- `acks=0`：不等待 broker 确认，吞吐高但可靠性低。
- `acks=1`：leader 写入成功就确认。
- `acks=all` 或 `-1`：等待 ISR 中副本确认，可靠性更高。

生产环境中，对于重要业务事件，通常选择 `acks=all`，再配合合适的 replication factor、min.insync.replicas 和 retries。

但要说明：

```text
acks=all 不等于绝对不丢消息，它只是提高 broker 写入确认级别。
```

业务端还要处理发送失败、超时、Outbox 和重试。

---

### 30. retries 有什么作用？会不会造成重复消息？

参考回答：

`retries` 表示 producer 发送失败时是否自动重试。

它可以提高临时网络抖动或 broker 短暂不可用时的发送成功率。

但重试可能带来问题：

- 应用层超时后，broker 实际已经写入成功。
- producer 再次发送，可能形成重复消息。
- 如果没有幂等 producer 或顺序控制，可能影响顺序。

所以生产中通常要：

- 开启幂等 producer。
- 业务事件带 event_id。
- consumer 做幂等处理。
- 重要业务使用 Outbox 确保事务边界。

---

### 31. 什么是幂等 Producer？

参考回答：

幂等 Producer 是 Kafka 为减少 producer 重试导致重复写入提供的机制。

开启后，producer 会为发送到 broker 的消息维护一些序列信息，broker 可以识别同一个 producer 对同一个 partition 的重复发送，从而避免重复追加。

它解决的是 producer 到 broker 之间因为重试导致的部分重复问题。

但要注意：

```text
幂等 Producer 不等于业务幂等。
```

consumer 仍然可能重复消费，业务仍然要通过唯一约束、processed_events 表、状态机等方式保证幂等。

---

### 32. batch.size 和 linger.ms 有什么作用？

参考回答：

`batch.size` 控制 producer 批次大小，`linger.ms` 控制为了凑批最多等待多久。

它们共同影响吞吐和延迟：

- batch 更大：吞吐可能更高，但内存占用可能增加。
- linger 更大：更容易凑批，但单条消息延迟可能增加。

如果业务是高吞吐日志或事件流，可以适当增加 batch 和 linger。

如果业务对实时性要求高，就不能盲目增大 linger。

面试中可以这样说：

```text
调 Producer 性能时不能只看吞吐，也要看 P95、P99 延迟和业务可接受范围。
```

---

### 33. compression 有什么作用？

参考回答：

compression 表示 producer 对消息批次进行压缩。

好处：

- 减少网络传输量。
- 减少 broker 磁盘占用。
- 在消息较大、重复字段较多时提升吞吐。

代价：

- producer 和 consumer 需要消耗 CPU 压缩和解压。
- 小消息或低流量场景收益可能不明显。

常见压缩算法包括 gzip、snappy、lz4、zstd 等。实际选择要压测。

---

### 34. Producer 发送成功后应该记录什么日志？

参考回答：

建议记录：

```text
event_id
topic
partition
offset
key
schema_version
trace_id
duration
```

这些日志有助于排查：

- 消息是否真的发出。
- 发到了哪个 partition。
- offset 是多少。
- consumer 是否消费到同一条事件。
- 是否存在重复发送。

不要只写一条“send success”，那对生产排障帮助很小。

---

### 35. Producer 发送失败时如何处理？

参考回答：

要先区分失败类型：

- 可重试错误，例如网络抖动、broker 短暂不可用。
- 不可重试错误，例如消息太大、序列化失败、认证失败。
- 模糊状态，例如客户端超时但 broker 可能已写入。

处理策略：

- 可重试错误可以重试。
- 不可重试错误应该记录并告警。
- 重要业务不能只依赖内存重试，应该考虑 Outbox。
- 事件必须有 event_id，方便后续幂等和排查。

对于订单创建这类重要业务，推荐：

```text
业务数据和 outbox 记录在同一个数据库事务中提交。
后台 publisher 扫描 outbox 并发送 Kafka。
发送成功后标记 published。
发送失败后记录错误并重试。
```

---

### 36. Producer 发送消息和数据库事务如何保证一致？

参考回答：

这是 Kafka 面试中的高频问题。

如果业务先写数据库，再发 Kafka，可能出现：

```text
数据库提交成功。
服务宕机。
Kafka 消息没发出去。
```

如果先发 Kafka，再写数据库，可能出现：

```text
Kafka 消息发出。
数据库事务回滚。
下游消费到并不存在的业务事实。
```

常见解决方案是 Outbox Pattern：

```text
在同一个数据库事务中写业务表和 outbox 表。
事务提交后，由后台任务扫描 outbox 表发送 Kafka。
发送成功后更新 outbox 状态。
失败则重试。
```

这样可以把“业务数据”和“待发送事件”放进同一个本地事务。

---

## 四、Consumer 消费者类

### 37. Consumer Group 是什么？

参考回答：

Consumer Group 是 Kafka 中实现消费扩展和消费进度隔离的机制。

同一个 group 内：

- 一个 partition 同一时间通常只会分配给一个 consumer。
- 多个 consumer 可以并行消费不同 partition。
- group 维护自己对每个 partition 的 offset。

不同 group 之间：

- 消费进度相互独立。
- 可以重复消费同一个 topic。

例如：

```text
inventory-service group 消费 order.created。
notification-service group 也消费 order.created。
两个 group 互不影响。
```

---

### 38. Consumer Group 和普通 Consumer 有什么区别？

参考回答：

普通说 consumer 是消费客户端，而 consumer group 是一组 consumer 共同消费 topic 的机制。

面试中可以这样解释：

```text
consumer group 解决的是多个实例如何协同消费、如何分配 partition、如何保存消费进度的问题。
```

Go 后端部署多个 consumer 实例时，一般会让它们使用同一个 group id。这样 Kafka 会把 partition 分配给这些实例，实现水平扩展。

---

### 39. 为什么同一个 group 里 consumer 数量超过 partition 数后，多出来的 consumer 不工作？

参考回答：

因为同一个 consumer group 内，一个 partition 同一时间只能由一个 consumer 消费。

如果 topic 有 3 个 partition，同一个 group 启动 10 个 consumer，最多只有 3 个 consumer 能分到 partition，剩下的会空闲。

所以 consumer 扩容前要看：

- topic partition 数。
- 当前 consumer 实例数。
- 每个 partition 的 lag。
- 单个 consumer 的处理能力。

不能看到 lag 高就盲目加实例。如果 partition 数不足，加实例也没用。

---

### 40. Kafka 如何保存消费进度？

参考回答：

Kafka 使用 offset 表示消费进度。

consumer group 会为每个 topic partition 保存已提交的 offset。现代 Kafka 中，这些 offset 存储在内部 topic `__consumer_offsets` 中。

需要注意：

```text
提交 offset 表示这个 group 认为某个 partition 的消息已经处理到哪里。
```

如果 offset 提交太早，业务失败后消息可能不会再被消费，造成业务丢失。

如果 offset 提交太晚，服务重启后可能重复消费。

所以真实业务中通常选择：

```text
业务处理成功后，再手动提交 offset。
```

---

### 41. 自动提交 offset 有什么风险？

参考回答：

自动提交 offset 的风险是：offset 可能在业务真正处理成功前就已经提交。

例如：

```text
consumer 拉到消息。
自动提交 offset。
业务写数据库失败。
服务崩溃。
重启后不会再消费这条消息。
```

这样就出现了业务意义上的消息丢失。

自动提交适合简单 demo 或对丢失不敏感的场景。对订单、支付、库存等业务，建议手动提交。

---

### 42. 手动提交 offset 的正确时机是什么？

参考回答：

一般原则是：

```text
业务处理成功后，再提交 offset。
```

更完整的流程：

```text
拉取消息
反序列化
校验
执行业务逻辑
业务成功
提交 offset
```

如果业务失败：

- 可重试错误：不要提交，进入重试策略。
- 不可恢复错误：写 DLQ 成功后，可以提交 offset，避免坏消息一直阻塞。
- 反序列化失败：通常写 DLQ 或错误表，然后提交 offset。

核心思想是：offset 提交必须和失败处理策略一起设计。

---

### 43. 什么是 Rebalance？

参考回答：

Rebalance 是 consumer group 重新分配 partition 的过程。

触发原因包括：

- group 中新增 consumer。
- consumer 下线或心跳超时。
- topic partition 数变化。
- consumer 长时间处理消息导致被认为失联。

Rebalance 期间可能出现：

- 暂停消费。
- partition 被重新分配。
- 短暂 lag 上升。
- 未提交 offset 的消息被其他 consumer 重新处理。

Go 服务中要特别注意优雅退出，避免正在处理的消息被中断。

---

### 44. 如何减少 Rebalance 的影响？

参考回答：

可以从几个方面处理：

- 合理设置 session timeout、heartbeat interval、max poll interval。
- 避免 handler 阻塞太久。
- 使用优雅退出，先停止拉取新消息，再等待处理中消息结束。
- 滚动发布时控制节奏。
- 避免频繁扩缩容。
- 监控 rebalance 次数和持续时间。

对于 Go 服务，退出流程建议是：

```text
收到 SIGTERM。
停止拉取新消息。
等待当前 handler 处理完。
提交成功消息的 offset。
关闭 consumer。
进程退出。
```

---

### 45. 什么是 Consumer Lag？

参考回答：

Consumer Lag 表示 consumer group 的消费进度落后于 partition 最新消息位置的程度。

简单理解：

```text
lag = partition 最新 offset - consumer group 已提交 offset
```

lag 高说明 consumer 处理速度跟不上 producer 写入速度，或者 consumer 出现故障。

但 lag 是结果，不是原因。排查时要继续看：

- 哪个 topic。
- 哪个 group。
- 哪个 partition。
- producer 写入速度是否上升。
- consumer 是否报错。
- handler 是否变慢。
- 是否发生 rebalance。
- 下游数据库或接口是否变慢。

---

### 46. lag 很高应该怎么排查？

参考回答：

排查顺序可以是：

1. 确认是哪个 topic、group、partition lag 高。
2. 查看 consumer 实例是否正常运行。
3. 查看消费失败率和错误日志。
4. 查看 handler 耗时是否上升。
5. 查看下游数据库、Redis、HTTP 接口是否变慢。
6. 查看是否发生 rebalance。
7. 查看 producer 写入速率是否突然增加。
8. 判断是否存在 key 热点导致单 partition lag 高。

如果只有一个 partition lag 很高，其他 partition 正常，很可能是：

- key 分布不均。
- 某类消息处理很慢。
- 对应 consumer 实例有问题。

如果所有 partition lag 都高，可能是整体消费能力不足或下游依赖变慢。

---

### 47. Consumer 处理消息时能不能并发？

参考回答：

可以，但要谨慎。

并发处理可以提升吞吐，但可能破坏同一个 partition 内的处理顺序。

如果业务不依赖顺序，可以在 consumer 内部使用 worker pool。但需要考虑：

- offset 如何提交。
- 某条消息失败时是否影响后续消息。
- 如何保证优雅退出。
- 如何限制并发避免打爆数据库。

如果业务要求同一个 key 或同一个订单严格顺序处理，就不能简单并发处理同一 partition 的消息。

---

### 48. Consumer 消费失败怎么办？

参考回答：

先分类：

- 临时错误：数据库超时、网络抖动、下游限流。
- 永久错误：消息格式错误、字段缺失、业务状态非法。
- 业务可预期失败：库存不足、订单已取消。

处理策略：

- 临时错误可以 retry。
- 多次 retry 失败后进入 DLQ。
- 永久错误直接进入 DLQ 或错误表。
- 业务可预期失败要看是否属于正常业务结果。

不要无限重试坏消息，否则会阻塞整个 partition。

---

### 49. 什么是 DLQ？

参考回答：

DLQ 是 Dead Letter Queue，死信队列。

当消息无法被正常处理，例如格式错误、重试多次仍失败，可以把消息写入 DLQ，避免它一直阻塞主消费链路。

DLQ 中通常要包含：

```text
原 topic
原 partition
原 offset
原 key
原 value
错误原因
失败时间
重试次数
trace_id
event_id
```

DLQ 不是垃圾桶。必须有：

- 告警。
- 查询工具。
- 人工修复方式。
- 受控重放机制。

---

### 50. DLQ 消息如何重放？

参考回答：

DLQ 重放要谨慎，不能直接把所有死信无脑发回主 topic。

重放前要确认：

- 错误原因是否已修复。
- schema 是否兼容。
- 重放是否会造成重复业务副作用。
- consumer 是否具备幂等能力。
- 是否需要限制重放速率。

建议重放工具支持：

```text
按 event_id 重放。
按时间范围重放。
按错误类型重放。
限速重放。
记录操作人和操作时间。
```

---

## 五、可靠性、幂等与一致性类

### 51. Kafka 能保证消息不丢吗？

参考回答：

不能简单说“Kafka 保证不丢消息”。

Kafka 可以通过配置提高可靠性，例如：

- replication factor。
- acks=all。
- min.insync.replicas。
- producer retries。
- 幂等 producer。
- 合理 retention。

但端到端不丢还取决于：

- producer 是否正确处理发送失败。
- 数据库事务和发消息是否一致。
- consumer 是否在业务成功后提交 offset。
- 失败消息是否有 retry 和 DLQ。
- 运维是否错误删除 topic 或缩短 retention。

更好的回答是：

```text
Kafka 提供了构建可靠消息链路的能力，但端到端可靠性需要 producer、broker、consumer 和业务幂等共同保证。
```

---

### 52. Kafka 的消息语义有哪些？

参考回答：

常见消息处理语义：

- at most once：最多一次，可能丢，不会重复。
- at least once：至少一次，不轻易丢，可能重复。
- exactly once：精确一次，限制条件较多。

大多数 Go 后端业务更常见的是：

```text
Kafka 链路使用 at-least-once，业务通过幂等保证重复消费不会产生重复副作用。
```

因为在真实系统里，网络超时、consumer 崩溃、rebalance、DLQ 重放都可能导致重复处理。

---

### 53. 什么是重复消费？

参考回答：

重复消费是指同一条业务事件被 consumer 处理多次。

常见原因：

- consumer 处理成功但提交 offset 前崩溃。
- offset 提交失败。
- rebalance 导致未提交消息被其他 consumer 接管。
- 人工重放 DLQ。
- producer 重复发送业务上等价的事件。

重复消费在 at-least-once 语义下是正常可能发生的情况。

所以正确做法不是假设不会重复，而是设计幂等。

---

### 54. 如何保证 Consumer 幂等？

参考回答：

常见方式：

1. 使用业务唯一键。
2. 使用 processed_events 表记录 event_id。
3. 使用数据库唯一约束。
4. 使用状态机防止状态倒退。
5. 使用乐观锁或版本号。

例如：

```sql
CREATE TABLE processed_events (
    event_id text PRIMARY KEY,
    processed_at timestamptz NOT NULL DEFAULT now()
);
```

处理消息时：

```text
开启数据库事务。
尝试插入 event_id。
如果唯一约束冲突，说明已经处理过，直接返回成功。
执行真正业务更新。
提交事务。
提交 Kafka offset。
```

这样即使消息重复到达，也不会重复扣库存或重复发放积分。

---

### 55. 幂等和去重有什么区别？

参考回答：

去重更偏技术动作：识别某条消息是否已经处理过。

幂等更偏业务语义：同一个操作执行一次或多次，最终业务结果一致。

例如：

- 通过 event_id 表跳过重复消息，这是去重。
- 订单状态从 CREATED 更新到 PAID，多次执行仍然是 PAID，这是幂等。
- 库存预占不能重复扣减，这是业务幂等。

真实项目中通常两者结合：

```text
用 event_id 做去重，用唯一约束和状态机保证业务幂等。
```

---

### 56. 消费成功后提交 offset 失败怎么办？

参考回答：

如果业务处理成功，但 offset 提交失败，那么 consumer 重启或 rebalance 后可能再次消费同一条消息。

这就是为什么 consumer 必须幂等。

处理思路：

- 业务成功写入数据库。
- offset 提交失败。
- 下次重复消费时，通过 event_id 或唯一约束识别已经处理。
- 直接返回成功并提交 offset。

所以不要把“提交 offset 成功”当成业务幂等的唯一保障。

---

### 57. 如果先提交 offset，再执行业务，会怎样？

参考回答：

这是危险做法。

可能出现：

```text
offset 已提交。
业务处理失败。
consumer 重启后不会再读这条消息。
业务结果丢失。
```

对于订单、支付、库存这类有副作用的消息，应该先业务处理成功，再提交 offset。

除非业务明确能接受丢失，比如某些非关键日志统计，否则不建议先提交 offset。

---

### 58. Kafka 事务是什么？Go 后端项目一定要用吗？

参考回答：

Kafka 事务用于让 producer 对多个 partition 的写入和 offset 提交等操作具备事务语义，常用于 Kafka Streams 或 consume-process-produce 场景。

但普通 Go 后端项目不一定必须使用 Kafka 事务。

原因：

- 业务通常还涉及数据库事务。
- Kafka 事务不能自动覆盖数据库本地事务。
- 实现复杂度较高。
- 大多数业务通过 Outbox + consumer 幂等已经能满足需求。

可以这样回答：

```text
如果只是数据库业务事件发布，我更倾向 Outbox Pattern；如果是纯 Kafka 内部的消费、处理、再生产链路，才会重点考虑 Kafka 事务。
```

---

### 59. Outbox Pattern 解决什么问题？

参考回答：

Outbox Pattern 解决的是数据库事务和消息发送之间的一致性问题。

问题是：

```text
业务数据写库成功，但 Kafka 消息发送失败。
或者 Kafka 消息发送成功，但业务事务回滚。
```

Outbox 做法：

```text
在同一个数据库事务中写业务表和 outbox 表。
后台 publisher 扫描 outbox。
发送 Kafka。
发送成功后标记 published。
失败则重试并记录错误。
```

这样至少保证：只要业务事务提交成功，待发送事件也一定落在数据库里，不会因为服务宕机而丢失。

---

### 60. Retry Topic 和 DLQ 有什么区别？

参考回答：

Retry Topic 用于暂存可恢复失败的消息，等待一段时间后再处理。

DLQ 用于存放无法正常处理或多次重试失败的消息。

区别：

| 类型 | 目的 | 消息状态 |
| --- | --- | --- |
| Retry Topic | 延迟后再试 | 仍有希望成功 |
| DLQ | 隔离坏消息 | 需要人工或工具处理 |

例如数据库短暂超时，可以进入 retry。JSON 缺少关键字段，通常直接进入 DLQ。

---

### 61. 如何设计 Retry？

参考回答：

Retry 要避免无限重试和立即重试。

常见设计：

- 记录 retry_count。
- 使用多个 retry topic 表示不同延迟级别。
- 使用指数退避。
- 达到最大次数后进入 DLQ。
- 保留原始 event_id 和错误原因。

例如：

```text
main topic -> retry 1min -> retry 5min -> retry 30min -> DLQ
```

同时 consumer 必须幂等，因为 retry 本质上也是重复投递。

---

## 六、Go 接入 Kafka 类

### 62. Go 中常见 Kafka 客户端有哪些？

参考回答：

Go 中常见 Kafka 客户端包括：

- `segmentio/kafka-go`
- `twmb/franz-go`
- `confluent-kafka-go`
- `Shopify/sarama` 或其社区维护分支

选择时要看：

- 是否支持 consumer group。
- 是否支持手动提交 offset。
- 是否方便配置 SASL/TLS。
- 是否容易获取 topic、partition、offset。
- 是否易于测试和封装。
- 项目维护状态。
- 团队熟悉程度。

面试中不要只说“我用某个库”。更好的回答是说明选择依据。

---

### 63. Go 项目中如何封装 Producer？

参考回答：

建议业务层依赖接口，而不是直接依赖 Kafka 客户端。

例如：

```go
type OrderEventPublisher interface {
    PublishOrderCreated(ctx context.Context, event OrderCreatedEvent) error
}
```

Kafka adapter 内部负责：

- 序列化事件。
- 设置 topic。
- 设置 key。
- 设置 headers。
- 调用客户端发送。
- 记录 topic、partition、offset。

这样业务 service 只关心“发布订单创建事件”，不关心 Kafka 底层细节。

好处：

- 方便单元测试。
- 方便替换客户端库。
- 代码边界清晰。

---

### 64. Go 项目中如何封装 Consumer？

参考回答：

Consumer 封装建议分为：

- 消费循环。
- 消息反序列化。
- 业务 handler。
- offset 提交。
- retry 和 DLQ。
- 优雅退出。

业务 handler 不应该直接提交 offset，也不应该直接操作 Kafka。

可以设计成：

```go
type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

外层 consumer 框架根据 error 类型决定：

- 成功：提交 offset。
- 可重试错误：进入 retry。
- 不可恢复错误：写 DLQ 后提交 offset。

---

### 65. Go Consumer 如何优雅退出？

参考回答：

优雅退出流程：

```text
监听 SIGTERM / SIGINT。
取消 context。
停止拉取新消息。
等待正在处理的消息完成。
成功处理的消息提交 offset。
关闭 consumer 客户端。
进程退出。
```

需要记录日志：

- 收到退出信号时间。
- 正在处理的消息数量。
- 最后提交的 offset。
- 退出耗时。
- 是否超时强制退出。

如果直接 kill 进程，可能导致处理成功但 offset 未提交，下一次重复消费。只要幂等做好，这不是灾难，但仍然要尽量优雅退出。

---

### 66. Go 中如何处理 Kafka 消息的 context？

参考回答：

Go 后端中 `context.Context` 主要用于：

- 控制发送超时。
- 控制消费处理超时。
- 传递 trace_id。
- 在服务退出时取消正在执行的操作。

例如 producer 发送时，应使用带超时的 context，避免请求无限阻塞。

consumer handler 中，也要把 context 传给数据库、HTTP 下游等调用。

但要注意：

```text
如果 context 超时导致业务处理结果不确定，consumer 必须依赖幂等来处理后续重复消费。
```

---

### 67. Go 中如何做 Kafka 单元测试？

参考回答：

不要所有测试都依赖真实 Kafka。

可以分层：

- 业务 service 测试：使用 fake publisher。
- consumer handler 测试：构造 fake message。
- producer adapter 测试：检查 key、headers、payload。
- 集成测试：再使用真实 Kafka 或 docker compose。

fake publisher 示例思路：

```go
type FakePublisher struct {
    Events []OrderCreatedEvent
}

func (f *FakePublisher) PublishOrderCreated(ctx context.Context, e OrderCreatedEvent) error {
    f.Events = append(f.Events, e)
    return nil
}
```

这样可以验证业务是否触发了事件发布，而不需要启动 Kafka。

---

### 68. Kafka 消息结构在 Go 中怎么设计？

参考回答：

建议事件结构包含：

```text
event_id
event_type
schema_version
occurred_at
trace_id
payload
```

订单创建事件示例：

```go
type OrderCreatedEvent struct {
    EventID       string    `json:"event_id"`
    EventType     string    `json:"event_type"`
    SchemaVersion int       `json:"schema_version"`
    OccurredAt    time.Time `json:"occurred_at"`
    TraceID       string    `json:"trace_id"`
    OrderID       string    `json:"order_id"`
    UserID        string    `json:"user_id"`
    Amount        int64     `json:"amount"`
}
```

这样下游可以做幂等、版本兼容、链路追踪和业务处理。

---

### 69. Kafka 配置在 Go 项目中应该如何管理？

参考回答：

配置应该集中管理，不要散落在业务代码里。

常见配置：

```text
brokers
topic
group_id
client_id
security_protocol
sasl_username
sasl_password
read_timeout
write_timeout
max_retries
```

启动时要校验必要配置。比如 broker、topic、group id 缺失时，程序应快速失败。

日志中可以打印 broker、topic、group id，但不要打印密码。

---

### 70. Go 服务中 Kafka 日志应该怎么打？

参考回答：

日志要能串起 producer 和 consumer。

Producer 日志：

```text
event_id
topic
partition
offset
key
send_duration
error
```

Consumer 日志：

```text
event_id
topic
partition
offset
group
handler_duration
commit_result
error
```

这样当某个订单状态异常时，可以按 `event_id` 或 `trace_id` 查完整链路。

---

## 七、性能调优与可观测性类

### 71. Kafka 性能调优应该从哪里开始？

参考回答：

不要一上来就改参数。先明确瓶颈在哪里：

- producer 发送慢。
- broker 写入慢。
- consumer 拉取慢。
- handler 处理慢。
- 下游数据库或接口慢。
- key 分布不均导致单 partition 热点。

调优流程：

```text
提出假设。
采集指标。
只改一个变量。
压测复现。
对比 P95/P99、吞吐、错误率和资源消耗。
形成结论。
```

---

### 72. Producer 侧有哪些性能参数？

参考回答：

常见参数：

- batch.size：批次大小。
- linger.ms：等待凑批时间。
- compression.type：压缩算法。
- buffer.memory：缓冲区大小。
- max.in.flight.requests.per.connection：并发请求数量。
- acks：确认级别，也影响延迟。

调优时要同时观察：

- 吞吐。
- 发送延迟。
- 错误率。
- CPU。
- 网络流量。
- batch 实际大小。

---

### 73. Consumer 侧有哪些性能参数？

参考回答：

常见关注点：

- 每次 poll 拉取多少消息。
- 拉取等待时间。
- 单条 handler 耗时。
- consumer 实例数。
- partition 数。
- 是否批量写数据库。
- 是否有并发处理。

如果 handler 平均耗时 20ms，单 consumer 串行大约每秒处理 50 条。若 producer 每秒写入 500 条，就需要优化 handler、增加并行度或增加 partition 后扩容 consumer。

---

### 74. 如何判断性能瓶颈在 Consumer 还是下游数据库？

参考回答：

看几个指标：

- consumer 拉取是否正常。
- handler duration 是否升高。
- 数据库慢查询是否增加。
- 数据库连接池是否耗尽。
- consumer 错误日志是否出现数据库超时。
- lag 是否和数据库延迟同步上升。

如果 Kafka 拉取很快，但 handler 大量卡在数据库写入，那瓶颈不是 Kafka，而是数据库或业务处理逻辑。

---

### 75. Kafka 监控应该关注哪些指标？

参考回答：

Producer：

- 发送速率。
- 发送错误率。
- 请求延迟。
- batch 大小。
- 重试次数。

Consumer：

- lag。
- 消费速率。
- handler duration。
- commit 失败次数。
- rebalance 次数。
- DLQ 数量。

Broker：

- 磁盘使用。
- 网络 IO。
- request latency。
- under replicated partitions。
- offline partitions。
- ISR shrink。

业务：

- 订单处理成功率。
- 库存预占成功率。
- 业务失败类型占比。

---

### 76. Prometheus 指标设计要注意什么？

参考回答：

最重要的是 label 控制。

不要把高基数字段放进 label，例如：

- order_id。
- user_id。
- event_id。
- trace_id。

这些字段取值无限增长，会导致 Prometheus 时间序列爆炸。

适合做 label 的字段：

- topic。
- group。
- result。
- error_type。
- service。

示例指标：

```text
kafka_consumer_messages_total{topic, group, result}
kafka_consumer_handler_duration_seconds{topic, group}
kafka_consumer_dlq_total{topic, group, error_type}
```

---

### 77. 如何设计 Kafka 告警？

参考回答：

常见告警：

- consumer lag 持续增长。
- DLQ 数量增加。
- 发送错误率升高。
- consumer 处理失败率升高。
- rebalance 频繁。
- broker 磁盘使用率过高。
- under replicated partitions 大于 0。
- offline partitions 大于 0。

告警不要只看瞬时值，要看持续时间和业务影响。

例如：

```text
lag 连续 10 分钟增长且没有回落。
DLQ 5 分钟内新增超过阈值。
offline partition 一出现就高优先级告警。
```

---

### 78. 如何做 Kafka 压测？

参考回答：

压测前要写清楚：

- Kafka 版本。
- broker 数量。
- topic partition 数。
- replication factor。
- 消息大小。
- key 分布。
- producer 参数。
- consumer 参数。
- 下游依赖是否 mock。
- 压测持续时间。

压测结果要记录：

- 吞吐。
- P95/P99 延迟。
- 错误率。
- 最大 lag。
- CPU、内存、网络、磁盘。
- 结论和复测方式。

不要只写“QPS 达到多少”。Kafka 链路要看端到端处理能力。

---

## 八、生产实践与架构设计类

### 79. 如何设计 topic？

参考回答：

topic 设计要考虑：

- 业务语义。
- 消息 schema。
- partition 数。
- replication factor。
- retention。
- 权限。
- 监控。
- 下游消费方。

例如订单事件：

```text
topic：order.created
key：order_id
partition：根据峰值流量和 consumer 并行度估算
replication factor：3
retention：3-7 天，视恢复要求决定
schema：包含 event_id、schema_version、occurred_at、order_id 等
```

---

### 80. 如何估算 partition 数？

参考回答：

可以从两个方向估算：

1. 写入吞吐需要多少 partition。
2. 消费处理需要多少并行度。

例如：

```text
峰值写入 1000 条/秒。
单个 consumer 每秒处理 100 条。
希望 10 个 consumer 并行处理。
则 partition 至少要接近 10。
```

还要留一定增长空间，但不能无限增加。

最终要通过压测验证，而不是只靠公式。

---

### 81. Kafka 如何做权限控制？

参考回答：

生产环境中要启用认证和授权。

常见做法：

- SASL 或 TLS 做认证。
- ACL 控制服务能读写哪些 topic。
- 不同服务使用不同账号。
- 运维重放工具单独授权。

例如：

```text
order-service 只能写 order.created。
inventory-service 只能读 order.created，写 inventory.reserved。
普通业务服务不能读取 DLQ。
```

这样可以降低误操作和越权访问风险。

---

### 82. Schema 如何演进？

参考回答：

事件 schema 一旦写入 Kafka，就会被多个下游消费，不能随便破坏兼容性。

建议：

- 增加字段时提供默认值。
- 不要轻易删除字段。
- 不要随意改变字段含义。
- 使用 schema_version。
- consumer 对未知字段保持兼容。
- 先升级 consumer，再升级 producer。
- 必要时使用新 topic 承载不兼容变更。

面试表达：

```text
Kafka schema 演进要默认存在新旧 producer 和新旧 consumer 并存的窗口期。
```

---

### 83. 事件驱动架构有什么优缺点？

参考回答：

优点：

- 服务解耦。
- 下游可异步处理。
- 更容易扩展多个订阅方。
- 能削峰填谷。
- 事件可以用于审计和重放。

缺点：

- 最终一致性更复杂。
- 排障链路更长。
- 重复消费必须处理。
- schema 演进要谨慎。
- 需要监控、告警、DLQ 和重放工具。

可以总结：

```text
事件驱动不是为了炫技，而是为了让系统边界更清楚、故障影响更可控。
```

---

### 84. Kafka 链路上线前要评审什么？

参考回答：

上线前至少评审：

- topic 是否创建。
- partition 和副本数是否合理。
- retention 是否满足恢复要求。
- producer 可靠性参数。
- consumer 是否手动提交 offset。
- consumer 是否幂等。
- retry 和 DLQ 是否设计。
- ACL 是否配置。
- 监控和告警是否就绪。
- 是否有回滚方案。
- 是否有重放工具或人工处理流程。

评审结论要具体，例如：

```text
有条件通过：上线前必须补充 DLQ 告警和重放权限控制。
```

---

### 85. Kafka 出现消息堆积时，生产上如何处理？

参考回答：

先判断堆积原因，不要直接扩容。

处理步骤：

1. 定位 topic、group、partition。
2. 看是否单 partition 热点。
3. 看 consumer 是否报错。
4. 看 handler 是否变慢。
5. 看下游依赖是否异常。
6. 临时扩 consumer 前确认 partition 数足够。
7. 必要时限流 producer。
8. 对坏消息进入 DLQ，避免阻塞主链路。
9. 恢复后观察 lag 是否回落。

如果堆积是数据库慢导致的，加 consumer 可能会把数据库打得更垮。

---

### 86. Kafka 中消息太大怎么办？

参考回答：

消息太大会带来：

- producer 发送失败。
- broker 内存和网络压力增大。
- consumer 拉取慢。
- 重试成本高。

处理方式：

- 消息只放必要字段。
- 大文件或大对象放对象存储，Kafka 只传引用。
- 压缩消息。
- 调整最大消息大小配置，但要谨慎。
- 检查 schema 是否过度膨胀。

面试中可以说：

```text
Kafka 更适合传事件和元数据，不适合直接传大文件。
```

---

## 九、项目场景追问类

### 87. 如果让你设计订单创建事件，你会怎么设计？

参考回答：

我会设计：

```text
topic：order.created
key：order_id
event_id：全局唯一，用于幂等
schema_version：用于兼容演进
occurred_at：事件发生时间
trace_id：链路追踪
payload：order_id、user_id、amount、items、created_at
```

Producer：

- 订单服务创建订单。
- 同一事务写 orders 表和 outbox 表。
- 后台 publisher 发送 Kafka。

Consumer：

- 库存服务消费。
- 使用 event_id 做幂等。
- 预占库存成功后提交 offset。
- 失败进入 retry 或 DLQ。

观测：

- 发送失败率。
- consumer lag。
- DLQ 数量。
- 库存预占成功率。

---

### 88. 订单服务写库成功但消息没发出去怎么办？

参考回答：

这是典型的数据库事务和消息发送一致性问题。

解决方案是 Outbox：

```text
订单服务在同一事务中写 orders 和 outbox_events。
事务提交后，即使服务崩溃，outbox 记录还在。
publisher 后台扫描 pending outbox 发送 Kafka。
发送成功后标记 published。
发送失败则重试。
```

这样不会因为进程宕机导致业务事件永久丢失。

---

### 89. 库存服务重复消费订单事件怎么办？

参考回答：

库存服务必须做幂等。

可以设计：

```sql
CREATE TABLE inventory_processed_events (
    event_id text PRIMARY KEY,
    order_id text NOT NULL,
    processed_at timestamptz NOT NULL DEFAULT now()
);
```

处理流程：

```text
开启事务。
插入 event_id。
如果冲突，说明处理过，直接返回成功。
扣减或预占库存。
提交事务。
提交 offset。
```

同时库存预占表可以对 `order_id` 加唯一约束，防止同一订单重复预占。

---

### 90. 如果库存不足，是不是要进 DLQ？

参考回答：

不一定。

库存不足可能是业务可预期结果，不一定是系统异常。

更合理的做法可能是：

- 发布 `inventory.reserve_failed` 事件。
- 订单服务消费后把订单改为失败或待处理。
- 不把它当成技术死信。

DLQ 更适合：

- 消息格式错误。
- schema 不兼容。
- 多次重试仍失败的系统错误。

面试中要区分业务失败和技术失败。

---

### 91. 下游服务挂了，Kafka 能解决什么，不能解决什么？

参考回答：

Kafka 能解决：

- 消息暂存。
- 上游和下游解耦。
- 下游恢复后继续消费。
- 一定程度削峰。

Kafka 不能自动解决：

- 下游业务 bug。
- 消息格式错误。
- 重复消费副作用。
- 无限增长的 lag。
- 数据库写入瓶颈。

所以还需要：

- consumer 幂等。
- retry 和 DLQ。
- lag 告警。
- 下游容量评估。
- 必要时限流上游。

---

### 92. 如果面试官问“Kafka 如何保证最终一致性”，怎么回答？

参考回答：

Kafka 本身不直接“保证业务最终一致性”，它提供可靠事件传递能力。最终一致性要靠业务链路设计。

以订单和库存为例：

```text
订单服务创建订单并写 outbox。
publisher 把 order.created 发到 Kafka。
库存服务消费事件并幂等预占库存。
预占成功后发布 inventory.reserved。
订单服务消费库存结果更新订单状态。
失败则走补偿或取消流程。
```

关键点：

- 事件必须可靠发布。
- consumer 必须幂等。
- 失败要有 retry 和 DLQ。
- 状态流转要有补偿。
- 监控要能发现卡住的状态。

---

### 93. 如果 Kafka 挂了，业务系统怎么办？

参考回答：

要看业务对 Kafka 的依赖方式。

如果使用 Outbox：

- 订单主流程可以先写数据库和 outbox。
- Kafka 不可用时，publisher 发送失败，outbox 保持 pending。
- Kafka 恢复后继续发送。
- 需要监控 outbox pending age 和 pending count。

如果业务请求中同步发送 Kafka：

- Kafka 不可用可能导致接口失败或超时。
- 需要降级策略。

对于关键业务，我会尽量避免在用户请求链路中强依赖 Kafka 发送成功，而是使用 Outbox 解耦。

---

### 94. 如果 Kafka 里有一条坏消息一直消费失败怎么办？

参考回答：

不能让坏消息无限阻塞 partition。

处理方式：

```text
识别错误类型。
如果是格式错误或不可恢复错误，写入 DLQ。
写 DLQ 成功后提交 offset。
触发告警。
人工修复后通过受控工具重放。
```

如果是临时错误，可以先进 retry topic，多次失败后再进 DLQ。

---

### 95. 你在简历里写 Kafka 项目，面试官可能怎么追问？

参考回答：

常见追问包括：

- 为什么项目要用 Kafka？
- 你设计了哪些 topic？
- partition 数怎么定？
- message key 用什么？
- 如何保证同一订单消息顺序？
- 如何处理重复消费？
- 失败消息怎么办？
- 是否使用 Outbox？
- consumer lag 怎么看？
- 服务发布时如何避免消息丢失？
- Kafka 不可用时系统如何处理？

准备简历项目时，一定要能把一条消息从 producer 讲到 consumer：

```text
HTTP 请求 -> 数据库事务 -> outbox -> Kafka -> consumer -> 幂等处理 -> offset 提交 -> 指标和日志
```

---

## 十、高频简答速记

### 96. topic、partition、offset 的关系是什么？

参考回答：

topic 是逻辑事件流，partition 是 topic 的物理分片，offset 是消息在某个 partition 内的位置。

一句话：

```text
topic 包含多个 partition，每个 partition 内的消息按 offset 顺序追加。
```

---

### 97. Kafka 保证全局有序吗？

参考回答：

不保证整个 topic 全局有序，只保证单个 partition 内有序。

如果需要某个业务维度有序，要用合适的 key 让同一维度进入同一个 partition。

---

### 98. 一个 partition 可以被同一个 group 的多个 consumer 同时消费吗？

参考回答：

通常不可以。

同一个 consumer group 内，一个 partition 同一时间只会分配给一个 consumer。这是 Kafka 保证 partition 内消费顺序的重要基础。

---

### 99. 不同 consumer group 会互相影响 offset 吗？

参考回答：

不会。

不同 consumer group 有独立的 offset，可以各自按自己的进度消费同一个 topic。

---

### 100. Kafka 消息会不会因为 consumer 消费了就删除？

参考回答：

不会。

Kafka 消息是否删除取决于 retention 或 compaction 策略，不取决于某个 consumer 是否消费过。

---

### 101. 为什么 consumer 需要幂等？

参考回答：

因为 Kafka 常见使用语义是 at-least-once，可能重复消费。

重复来源包括 consumer 崩溃、提交 offset 失败、rebalance、DLQ 重放等。

---

### 102. 为什么要用 event_id？

参考回答：

event_id 用于唯一标识一条业务事件，方便：

- 幂等处理。
- 日志追踪。
- DLQ 排查。
- 人工重放。
- 跨服务链路关联。

---

### 103. Kafka 适合传大文件吗？

参考回答：

不适合。

更好的方式是大文件放对象存储，Kafka 只传文件 URL、对象 key 或元数据。

---

### 104. 什么情况下要用 DLQ？

参考回答：

当消息无法正常处理，或者重试多次仍失败时，应该进入 DLQ，避免阻塞主消费链路。

---

### 105. 什么情况下不要用 DLQ？

参考回答：

正常业务失败不一定要进 DLQ，例如库存不足、优惠券不可用。这些应该建模为业务结果，而不是技术死信。

---

### 106. Kafka 的 Exactly Once 是否能解决所有重复问题？

参考回答：

不能。

Exactly Once 有适用边界，主要解决 Kafka 内部某些生产和消费处理场景。涉及外部数据库、HTTP 接口、人工重放时，仍然需要业务幂等。

---

### 107. 面试中如何一句话解释 Outbox？

参考回答：

Outbox 是把业务数据和待发送事件写入同一个数据库事务，再由后台任务可靠发送 Kafka，用来解决数据库事务和消息发送之间的一致性缺口。

---

### 108. 面试中如何一句话解释 Consumer Lag？

参考回答：

Consumer Lag 是 consumer group 已提交 offset 落后于 partition 最新 offset 的数量，用来衡量消费积压。

---

### 109. 面试中如何一句话解释 Rebalance？

参考回答：

Rebalance 是 consumer group 成员变化或 partition 变化时，Kafka 重新分配 partition 给 consumer 的过程。

---

### 110. 面试中如何回答“你怎么保证 Kafka 链路可靠”？

参考回答：

可以按端到端回答：

```text
Producer 侧使用合理 acks、retries、幂等 producer，并通过 Outbox 保证业务数据和事件发布一致。
Broker 侧使用多副本、ISR 和合理 retention。
Consumer 侧手动提交 offset，业务成功后再提交。
业务侧通过 event_id、唯一约束和状态机保证幂等。
失败侧设计 retry、DLQ 和重放。
观测侧监控 lag、失败率、DLQ、outbox pending age 和发送错误率。
```

这比只说“acks=all”完整得多。

---

## 十一、面试表达模板

### 111. 如果被问“你项目里 Kafka 怎么用的”，可以这样答

参考回答：

我在项目里主要把 Kafka 用在订单事件驱动链路中。

订单服务创建订单后，会生成 `order.created` 事件。为了避免数据库事务成功但消息发送失败，我没有直接在请求里同步发 Kafka，而是使用 Outbox Pattern：订单表和 outbox 表在同一个事务里写入。后台 publisher 扫描 outbox，把事件发送到 Kafka，发送成功后标记 published。

事件的 key 使用 `order_id`，这样同一个订单的相关事件可以进入同一个 partition，保证订单维度的局部顺序。事件里包含 `event_id`、`schema_version`、`trace_id` 和 `occurred_at`。

库存服务作为 consumer group 消费 `order.created`，处理时先用 `event_id` 做幂等，业务成功后再手动提交 offset。如果处理失败，会按错误类型进入 retry 或 DLQ。链路上会监控 consumer lag、DLQ 数量、发送失败率和 outbox pending age。

这样设计的重点是：上游和下游解耦，同时通过 Outbox、幂等、手动提交和可观测性保证链路可靠。

---

### 112. 如果被问“Kafka 最大的坑是什么”，可以这样答

参考回答：

我觉得 Kafka 最大的坑不是会不会发消息，而是很多人低估了异步系统的复杂性。

常见坑包括：

- 以为发到 Kafka 就一定不会丢。
- 以为 consumer 不会重复消费。
- offset 自动提交导致业务失败后消息被跳过。
- 没有 DLQ，坏消息一直阻塞 partition。
- 没有 event_id，重复和重放无法追踪。
- topic 和 schema 设计混乱。
- lag 告警缺失，堆积很久才发现。

所以我会把 Kafka 链路当成一个完整系统设计，而不是只写 producer 和 consumer 代码。

---

### 113. 如果被问“你如何排查 Kafka 消息丢失”，可以这样答

参考回答：

我会先澄清“丢失”发生在哪个阶段：

```text
业务是否生成事件？
outbox 是否写入？
producer 是否发送成功？
broker 中是否能查到对应 topic、partition、offset？
consumer 是否拉取到？
handler 是否处理成功？
offset 是否提前提交？
消息是否进入 retry 或 DLQ？
retention 是否已经清理？
```

如果有 event_id，就可以沿着日志和表逐段追踪。

很多所谓消息丢失，其实是：

- producer 没发成功。
- consumer 提前提交 offset。
- 消息进了 DLQ 但没人看。
- retention 到期。
- 查错了 consumer group。

---

### 114. 如果被问“如何从 0 到 1 上线一条 Kafka 链路”，可以这样答

参考回答：

我会按这个流程：

```text
1. 明确业务事件和上下游。
2. 设计 topic、key、schema 和版本。
3. 估算 partition、retention 和副本数。
4. 设计 producer 可靠性策略。
5. 判断是否需要 Outbox。
6. 设计 consumer 幂等和手动提交。
7. 设计 retry、DLQ 和重放。
8. 配置 ACL。
9. 建立日志、指标和告警。
10. 做压测和故障演练。
11. 灰度上线。
12. 观察 lag、失败率和 DLQ。
```

这能体现你不是只会写代码，也理解生产上线。

---

## 十二、最后复习清单

面试前确认自己能回答这些问题：

```text
Kafka 为什么适合事件驱动？
topic、partition、offset、broker、replica 分别是什么？
Kafka 为什么只能保证 partition 内有序？
message key 怎么设计？
acks、retries、幂等 producer 分别解决什么问题？
Consumer Group 如何分配 partition？
offset 什么时候提交？
为什么会重复消费？
如何做业务幂等？
Outbox 解决什么问题？
Retry Topic 和 DLQ 如何设计？
Consumer Lag 怎么排查？
Go 服务如何封装 Producer 和 Consumer？
Go Consumer 如何优雅退出？
Kafka 链路上线前要评审什么？
项目里如何讲清一条消息的完整生命周期？
```

如果这些问题都能结合订单项目讲清楚，Kafka 面试基本就不会只停留在背概念了。
