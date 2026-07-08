# Kafka 速查表

这份速查表用于面试前快速复习。

它不替代系统教程，只帮你在短时间内想起：

```text
概念是什么。
面试怎么说。
容易踩什么坑。
Go 后端项目里落到哪里。
```

---

## 一、核心概念速查

| 概念 | 一句话解释 | 面试关键词 | 常见坑 |
| --- | --- | --- | --- |
| Kafka | 分布式事件流平台 | 异步解耦、削峰填谷、事件驱动、可重放 | 不要说成“万能消息队列” |
| Broker | Kafka 集群中的服务节点 | 存储 partition、处理读写、复制数据 | broker 不是 topic |
| Topic | 一类消息或事件流的逻辑名称 | `order.created`、业务语义、权限边界 | topic 过大过杂会导致 schema 混乱 |
| Partition | topic 的物理分片 | 并行度、顺序追加、水平扩展 | Kafka 只保证 partition 内有序 |
| Offset | 消息在 partition 内的位置 | 消费进度、提交 offset、lag | offset 不是 topic 全局编号 |
| Replica | partition 的副本 | 高可用、容错、leader/follower | 副本不能提升同一 group 的消费并行度 |
| Leader | partition 的主副本 | producer 写入、consumer 读取 | leader 挂了需要重新选举 |
| Follower | partition 的从副本 | 从 leader 同步数据 | 落后太多会影响 ISR |
| ISR | 同步副本集合 | `acks=all`、可靠性、ISR shrink | ISR 不是所有副本 |
| Consumer Group | 一组 consumer 协同消费 | 扩展消费、独立 offset、多订阅 | group 内一个 partition 同时只给一个 consumer |

---

## 二、Topic / Partition / Offset

| 问题 | 快速回答 |
| --- | --- |
| topic 是什么？ | topic 是消息的逻辑分类，通常对应一类业务事件，例如 `order.created`。 |
| partition 是什么？ | partition 是 topic 的物理分片，每个 partition 内消息按 offset 顺序追加。 |
| offset 是什么？ | offset 是消息在某个 partition 内的位置，只在该 partition 内有意义。 |
| Kafka 保证全局有序吗？ | 不保证 topic 全局有序，只保证单个 partition 内有序。 |
| 如何保证同一订单有序？ | 用 `order_id` 作为 message key，让同一订单进入同一个 partition。 |
| partition 越多越好吗？ | 不是。partition 多能提升并行度，但会增加 broker 元数据、文件、rebalance 成本。 |
| consumer 数量超过 partition 数会怎样？ | 同一 group 内多出来的 consumer 会空闲。 |
| 增加 partition 有什么风险？ | key 到 partition 的映射可能变化，可能影响历史顺序假设，并触发 rebalance。 |

记忆句：

```text
topic 是逻辑事件流，partition 是物理分片，offset 是 partition 内的位置。
```

---

## 三、Producer 速查

| 配置 / 概念 | 作用 | 面试说法 | 常见坑 |
| --- | --- | --- | --- |
| key | 决定消息进入哪个 partition | 用业务 key 保证局部顺序 | key 选择不当会导致热点 |
| value | 消息正文 | 通常是 JSON、Avro、Protobuf 等 | 不要塞大文件 |
| headers | 消息元数据 | 放 `trace_id`、`event_id`、`schema_version` | 不要放无限增长的大字段 |
| `acks=0` | 不等待确认 | 吞吐高，可靠性低 | 重要业务不要用 |
| `acks=1` | leader 写入成功即确认 | 折中方案 | leader 崩溃时可能丢已确认消息 |
| `acks=all` | 等待 ISR 确认 | 重要业务常用 | 不等于绝对不丢 |
| retries | 发送失败自动重试 | 应对临时故障 | 可能造成重复，需要幂等 |
| 幂等 Producer | 减少 producer 重试导致的重复写入 | Producer 到 Broker 侧幂等 | 不等于业务幂等 |
| batch.size | 批量大小 | 提升吞吐 | 过大可能增加延迟和内存 |
| linger.ms | 等待凑批时间 | 吞吐和延迟的取舍 | 过大会增加尾延迟 |
| compression | 压缩消息批次 | 降低网络和磁盘 | 会增加 CPU 成本 |

Producer 面试模板：

```text
重要业务事件通常使用 acks=all、retries、幂等 Producer。
但这只解决 producer 到 broker 的一部分可靠性。
如果还涉及数据库事务，我会使用 Outbox Pattern。
```

---

## 四、Message Key 速查

| key 选择 | 适合场景 | 风险 |
| --- | --- | --- |
| `order_id` | 订单维度顺序 | 单个超高频订单较少，通常较稳 |
| `user_id` | 用户维度顺序 | 大用户可能形成热点 |
| `shop_id` | 店铺维度顺序 | 大商家容易压到单 partition |
| 空 key | 不关心业务顺序 | 无法保证某个业务维度有序 |
| 随机 key | 追求均匀分布 | 破坏业务维度顺序 |

判断公式：

```text
业务需要哪个维度有序，就优先考虑哪个维度作为 key。
但同时要评估这个 key 会不会导致热点。
```

---

## 五、Consumer Group 速查

| 问题 | 快速回答 |
| --- | --- |
| Consumer Group 解决什么？ | 解决多个 consumer 如何协同消费、如何分配 partition、如何保存 offset。 |
| 同一 group 内一个 partition 能给多个 consumer 吗？ | 通常不能，同一时刻一个 partition 只给一个 consumer。 |
| 不同 group 会互相影响吗？ | 不会，不同 group 有独立 offset。 |
| group id 重要吗？ | 很重要，同一个 group id 表示共享消费进度。 |
| 新 group 从哪里开始消费？ | 取决于是否有已提交 offset，以及 `auto.offset.reset` 配置。 |
| 为什么加 consumer 没效果？ | 可能 consumer 数量已经超过 partition 数，或者瓶颈在数据库。 |

记忆句：

```text
partition 决定同一 group 的最大消费并行度。
```

---

## 六、Offset 提交速查

| 提交方式 | 特点 | 适合场景 | 风险 |
| --- | --- | --- | --- |
| 自动提交 | 客户端定期提交 offset | demo、非关键日志 | 可能业务失败但 offset 已提交 |
| 手动提交 | 业务成功后显式提交 | 订单、支付、库存等关键业务 | 代码复杂度更高 |
| 先提交再处理 | 延迟低 | 极少数可丢场景 | 业务失败后消息不会再来 |
| 先处理再提交 | 更可靠 | 大多数业务系统 | 可能重复消费，需要幂等 |

推荐流程：

```text
拉取消息
-> 反序列化
-> 校验
-> 执行业务逻辑
-> 业务成功
-> 提交 offset
```

失败处理：

```text
可重试错误：进入 retry，不提交或按策略处理。
不可恢复错误：写 DLQ 成功后提交 offset。
业务成功：提交 offset。
```

---

## 七、Rebalance 速查

| 项目 | 内容 |
| --- | --- |
| 是什么 | consumer group 重新分配 partition 的过程 |
| 触发原因 | consumer 加入、退出、心跳超时、partition 变化、处理太久 |
| 影响 | 消费暂停、lag 上升、未提交消息重复处理 |
| Go 服务重点 | 优雅退出、控制 handler 耗时、避免频繁扩缩容 |
| 监控指标 | rebalance 次数、rebalance 耗时、lag 变化 |

Go Consumer 优雅退出：

```text
收到 SIGTERM
-> 停止拉取新消息
-> 等待正在处理的消息完成
-> 成功后提交 offset
-> 关闭 consumer
-> 退出进程
```

面试句：

```text
Rebalance 本身不是异常，但频繁 rebalance 会导致消费停顿、lag 增长和重复消费风险。
```

---

## 八、Consumer Lag 速查

| 项目 | 内容 |
| --- | --- |
| 定义 | consumer group 已提交 offset 落后于 partition 最新 offset 的数量 |
| 公式 | `lag = log end offset - committed offset` |
| 说明 | lag 高表示消费积压，但 lag 是结果，不是原因 |
| 常见原因 | consumer 挂了、handler 慢、下游慢、producer 流量暴涨、key 热点、rebalance |

排查顺序：

```text
1. 确认 topic / group / partition。
2. 看是所有 partition lag 高，还是单 partition lag 高。
3. 看 consumer 是否运行。
4. 看错误日志和失败率。
5. 看 handler duration。
6. 看数据库、Redis、HTTP 下游是否变慢。
7. 看 producer 写入速率。
8. 看是否 rebalance。
9. 看 key 是否倾斜。
```

判断：

| 现象 | 可能原因 |
| --- | --- |
| 所有 partition lag 都高 | 整体消费能力不足、下游慢、producer 流量暴涨 |
| 单个 partition lag 高 | key 热点、某类消息慢、对应 consumer 异常 |
| lag 高但 consumer 无错误 | handler 慢、批量太小、下游瓶颈 |
| lag 周期性升高 | 定时流量、批处理任务、发布引发 rebalance |

---

## 九、可靠性速查

| 问题 | 关键答案 |
| --- | --- |
| Kafka 能保证不丢吗？ | Kafka 提供可靠机制，但端到端不丢要靠 Producer、Broker、Consumer 和业务共同保证。 |
| `acks=all` 是否足够？ | 不够，它只提高 broker 写入确认级别。 |
| 为什么会重复消费？ | 处理成功但提交 offset 前崩溃、offset 提交失败、rebalance、DLQ 重放等。 |
| 如何解决重复消费？ | 使用 event_id、唯一约束、processed_events 表、状态机。 |
| 业务成功但 offset 提交失败怎么办？ | 下次会重复消费，依赖幂等跳过或安全重入。 |
| 先提交 offset 再处理业务可以吗？ | 关键业务不建议，可能业务失败但消息被跳过。 |

端到端可靠性模板：

```text
Producer：acks=all、retries、幂等 Producer、event_id、Outbox。
Broker：多副本、ISR、合理 retention。
Consumer：手动提交 offset、业务成功后提交。
业务：唯一约束、processed_events、状态机。
失败：Retry、DLQ、重放工具。
观测：lag、DLQ、失败率、outbox pending age。
```

---

## 十、幂等速查

| 手段 | 说明 | 适合场景 |
| --- | --- | --- |
| event_id 去重 | 每个事件全局唯一 | 通用 |
| processed_events 表 | 记录已处理事件 | 消费端幂等 |
| 唯一约束 | 数据库强约束 | 订单预占、支付单、防重复写 |
| 状态机 | 防止状态倒退 | 订单状态、支付状态 |
| 版本号 | 防止旧事件覆盖新状态 | 用户资料、库存快照 |

典型流程：

```text
开启事务。
插入 event_id 到 processed_events。
如果唯一冲突，说明处理过，直接返回成功。
执行业务更新。
提交事务。
提交 offset。
```

面试句：

```text
Kafka 常见语义是 at-least-once，所以我默认 consumer 可能重复消费，业务必须幂等。
```

---

## 十一、Retry / DLQ 速查

| 概念 | 作用 | 注意点 |
| --- | --- | --- |
| Retry | 临时失败后延迟重试 | 不要无限立即重试 |
| Retry Topic | 存放等待重试的消息 | 可设计 1min、5min、30min 多级 |
| DLQ | 存放无法正常处理的消息 | 不是垃圾桶，需要告警和重放 |
| 重放 | 修复后重新投递 DLQ 消息 | 必须限速、可审计、幂等 |

错误分类：

| 错误类型 | 示例 | 策略 |
| --- | --- | --- |
| 临时错误 | 数据库超时、HTTP 429 | retry，多次失败后 DLQ |
| 不可恢复错误 | JSON 缺字段、schema 不兼容 | DLQ |
| 业务结果 | 库存不足、订单已取消 | 建模为业务事件，不一定进 DLQ |

DLQ 必备字段：

```text
原 topic
原 partition
原 offset
原 key
原 value
event_id
trace_id
错误原因
重试次数
失败时间
```

面试句：

```text
DLQ 不是最终处理结果，而是把坏消息隔离出来，避免阻塞主链路，并等待人工修复或受控重放。
```

---

## 十二、Outbox 速查

| 项目 | 内容 |
| --- | --- |
| 解决问题 | 数据库事务和 Kafka 发送之间的一致性缺口 |
| 核心做法 | 业务表和 outbox 表在同一事务写入 |
| 后台任务 | 扫描 pending outbox，发送 Kafka，成功后标记 published |
| 仍需幂等 | 发送成功但更新 outbox 失败时可能重复发送 |
| 观测指标 | pending count、pending age、publish failure rate |

Outbox 流程：

```text
开启数据库事务
-> 写业务表
-> 写 outbox_events
-> 提交事务
-> publisher 扫描 outbox
-> 发送 Kafka
-> 标记 published
```

面试句：

```text
Outbox 保证业务数据和待发送事件一起落库，Consumer 幂等负责处理可能的重复投递。
```

---

## 十三、Exactly Once 速查

| 问题 | 快速回答 |
| --- | --- |
| Kafka 支持 Exactly Once 吗？ | 支持特定边界内的 Exactly Once，例如 Kafka 内部事务和流处理场景。 |
| Go 业务项目还需要幂等吗？ | 需要。因为业务通常会写外部数据库、调用 HTTP 服务。 |
| Exactly Once 能保证 PostgreSQL 只更新一次吗？ | 不能自动保证，仍需业务幂等。 |
| 常见实践是什么？ | at-least-once + 业务幂等 + Outbox + Retry/DLQ。 |

面试句：

```text
Exactly Once 有适用边界，涉及外部数据库和业务副作用时，仍然要做业务幂等。
```

---

## 十四、性能参数速查

### Producer

| 参数 | 作用 | 取舍 |
| --- | --- | --- |
| `batch.size` | 批次大小 | 吞吐 vs 内存 |
| `linger.ms` | 等待凑批时间 | 吞吐 vs 延迟 |
| `compression.type` | 压缩算法 | 网络/磁盘 vs CPU |
| `buffer.memory` | 发送缓冲区 | 吞吐 vs 内存 |
| `acks` | 确认级别 | 可靠性 vs 延迟 |
| `retries` | 自动重试 | 成功率 vs 重复风险 |

### Consumer

| 参数 / 因素 | 作用 | 取舍 |
| --- | --- | --- |
| 拉取批次大小 | 每次拉多少消息 | 吞吐 vs 单批处理时间 |
| handler 耗时 | 单条消息业务处理时间 | 直接决定消费能力 |
| consumer 数量 | 并行消费实例 | 受 partition 数限制 |
| partition 数 | 最大并行度基础 | 过多增加运维成本 |
| 批量写库 | 减少数据库往返 | 失败处理更复杂 |

调优原则：

```text
先定位瓶颈，再改参数。
一次只改一个变量。
同时看吞吐、P95、P99、错误率、CPU、网络、lag。
```

---

## 十五、监控告警速查

| 类型 | 指标 |
| --- | --- |
| Producer | send rate、send error rate、request latency、retry count |
| Consumer | lag、consume rate、handler duration、commit failure、rebalance count |
| Broker | disk usage、network IO、under replicated partitions、offline partitions、ISR shrink |
| 业务 | 订单处理成功率、库存预占成功率、DLQ 业务错误分布 |
| Outbox | pending count、pending age、publish failure rate |

告警建议：

```text
offline partition：高优先级，立即告警。
under replicated partition：持续出现要告警。
consumer lag：持续增长且不回落才告警。
DLQ：新增超过阈值要告警。
outbox pending age：超过业务可接受延迟要告警。
```

Prometheus label 注意：

```text
可以做 label：topic、group、result、error_type、service。
不要做 label：order_id、user_id、event_id、trace_id。
```

---

## 十六、Go 后端落点速查

| 模块 | 负责什么 |
| --- | --- |
| config | broker、topic、group、超时、认证配置 |
| event | 事件结构、event_id、schema_version、序列化 |
| producer | 封装发送，不让业务直接依赖 Kafka 客户端 |
| consumer | 拉取消息、提交 offset、retry、DLQ、优雅退出 |
| handler | 只处理业务逻辑 |
| repository | 写数据库、幂等表、状态更新 |
| observability | 日志、指标、trace |

推荐接口：

```go
type OrderEventPublisher interface {
    PublishOrderCreated(ctx context.Context, event OrderCreatedEvent) error
}
```

Consumer 边界：

```text
handler 只返回成功或错误。
外层 consumer 根据错误类型决定 commit、retry 或 DLQ。
```

---

## 十七、上线评审速查

Kafka 链路上线前检查：

```text
topic 是否创建。
partition 数是否评估。
replication factor 是否合理。
retention 是否满足恢复要求。
message key 是否确定。
schema 是否包含 event_id、schema_version、occurred_at。
producer 是否有失败处理。
是否需要 Outbox。
consumer 是否手动提交 offset。
consumer 是否幂等。
retry 和 DLQ 是否设计。
DLQ 是否有告警和重放工具。
ACL 是否最小权限。
lag、失败率、DLQ、outbox pending 是否有指标。
是否做过压测。
是否有灰度和回滚方案。
```

---

## 十八、面试万能回答结构

遇到 Kafka 开放题，可以按这个结构答：

```text
1. 先解释概念。
2. 再说明解决什么问题。
3. 结合订单或库存项目举例。
4. 说出风险。
5. 给出兜底方案。
6. 最后补充如何观测。
```

示例：

```text
Kafka 在我的项目里用于订单事件驱动。
订单服务创建订单后发布 order.created。
为了避免数据库成功但消息丢失，我用 Outbox。
为了处理重复消费，consumer 使用 event_id 和唯一约束做幂等。
失败消息按错误类型进入 retry 或 DLQ。
线上通过 lag、DLQ、发送失败率和 outbox pending age 观察链路。
```

---

## 十九、最容易被追问的 20 个点

```text
1. Kafka 为什么适合事件驱动？
2. Kafka 和 RabbitMQ 有什么区别？
3. topic、partition、offset 的关系是什么？
4. Kafka 是否保证全局有序？
5. message key 如何设计？
6. partition 数怎么估算？
7. acks=all 是否能保证不丢？
8. 幂等 Producer 解决什么？
9. Consumer Group 如何分配 partition？
10. offset 什么时候提交？
11. 为什么会重复消费？
12. Consumer 幂等如何做？
13. Rebalance 是什么？
14. Consumer Lag 如何排查？
15. Retry 和 DLQ 有什么区别？
16. Outbox 解决什么问题？
17. Exactly Once 是否还需要业务幂等？
18. Go 项目如何封装 Producer / Consumer？
19. Kafka 不可用时业务怎么办？
20. 一条订单事件从产生到消费的完整链路是什么？
```

---

## 二十、最后一分钟速记

```text
Kafka：分布式事件流平台。
Topic：事件流名称。
Partition：并行和顺序的基本单位。
Offset：partition 内的位置。
Key：决定 partition，影响顺序和热点。
acks=all：提高 broker 确认级别，不等于绝对不丢。
Consumer Group：一组 consumer 协同消费，offset 独立。
手动提交：业务成功后提交 offset。
重复消费：正常可能发生，靠业务幂等。
Rebalance：重新分配 partition，可能导致短暂停顿和重复处理。
Lag：消费进度落后，是现象不是原因。
Retry：临时失败后再试。
DLQ：隔离坏消息，不是垃圾桶。
Outbox：解决数据库事务和消息发送一致性缺口。
Exactly Once：有边界，外部数据库仍需幂等。
Go 封装：业务依赖接口，Kafka 放 adapter。
生产上线：topic、schema、幂等、DLQ、监控、ACL、回滚都要评审。
```
