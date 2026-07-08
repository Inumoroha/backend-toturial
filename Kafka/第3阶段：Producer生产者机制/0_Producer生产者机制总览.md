# 第 3 阶段：Producer 生产者机制总览

这一阶段学习 Kafka producer。Producer 不是简单的“调用 send 方法”。在真实 Go 后端项目里，producer 决定消息是否可能丢、是否可能重复、吞吐高不高、延迟低不低、同一业务对象是否有序。

---

## 一、本阶段课程

1. `1_Producer发送流程.md`
2. `2_MessageKey与Partition选择.md`
3. `3_acks_retries与消息可靠性.md`
4. `4_batch_linger_compression与吞吐.md`
5. `5_幂等Producer与顺序性.md`
6. `6_综合实践_批量发送订单事件.md`

---

## 二、本阶段学习目标

完成后你应该能做到：

- 解释 producer 从发送到 broker 确认的流程。
- 为订单、支付、库存事件选择合适 message key。
- 理解 `acks=0`、`acks=1`、`acks=all`。
- 理解 retry 为什么可能导致重复消息。
- 理解 batch、linger、compression 如何影响吞吐和延迟。
- 理解幂等 producer 解决什么问题、不解决什么问题。

---

## 三、本阶段 Go 后端视角

Go 项目里 producer 应该统一封装：

```text
internal/kafka/producer.go
```

业务代码不应该到处直接调用 Kafka 客户端。统一封装可以保证：

- 所有消息都有 event_id。
- 所有发送都有 context timeout。
- 所有失败都有日志。
- 关键业务统一使用可靠配置。
- 服务退出时统一 flush 或 close。

---

## 四、本阶段最终产出

本阶段结束后，你应该能设计并解释一条生产级发送链路：

```text
业务 service
  构造 OrderCreatedEvent
  选择 message key = order_id
  设置 event_id/version/occurred_at
  调用 producer wrapper

producer wrapper
  设置 acks
  设置 retry
  设置 batch/linger/compression
  发送失败返回错误
```

至少要能写出这样的事件：

```json
{
  "event_id": "evt_order_created_order_1001",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-05T10:00:00Z",
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88"
  }
}
```

并且能说明为什么 key 使用 `order_id`，而不是随机字符串。

---

## 五、最小实验闭环

本阶段至少要完成：

```text
创建 order.created topic
-> 使用 key 发送多条订单事件
-> 使用 console consumer 打印 key
-> 观察同一 key 的消息进入同一 partition
-> 调整 batch/linger/compression
-> 对比吞吐和延迟
```

验证命令：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true
```

观察重点：

- 同一个 key 是否稳定落到同一个 partition。
- producer 发送失败时是否能返回错误。
- 批量参数改变后吞吐是否变化。

---

## 六、可靠性取舍

Producer 参数不是越大越好，也不是越快越好。

| 目标 | 常见配置方向 | 代价 |
| --- | --- | --- |
| 更可靠 | `acks=all`、重试、幂等 | 延迟可能增加 |
| 更高吞吐 | batch、linger、compression | 单条消息延迟可能增加 |
| 更低延迟 | 小 batch、低 linger | 吞吐可能下降 |
| 保持顺序 | 稳定 key、控制并发 | 并行度受限制 |

学习时要形成一个判断：

```text
关键业务先保证可靠性，再优化吞吐。
日志分析类业务可以更偏吞吐。
```

---

## 七、Go 后端常见错误

### 1. 不设置 message key

不设置 key 会让同一订单的事件分散到不同 partition，后续顺序会更难保证。

### 2. 忽略发送错误

关键业务里 `_ = producer.Send(...)` 是危险写法。发送失败必须返回错误或写入补偿表。

### 3. 以为 producer 幂等等于业务幂等

producer 幂等只能减少 producer retry 造成的重复写入，不能解决 consumer 重复处理、outbox 重发和 DLQ 重放。

### 4. 为了吞吐盲目增大 linger

linger 增大可以提升批量效果，但会增加消息等待时间。订单链路要评估用户可接受延迟。

---

## 八、本阶段验收

完成本阶段后，你应该能回答：

1. `acks=all` 比 `acks=1` 多保证了什么？
2. producer retry 为什么可能制造重复消息？
3. batch 和 linger 分别影响什么？
4. compression 为什么能提升吞吐？代价是什么？
5. 幂等 producer 解决什么，不解决什么？
6. 订单事件为什么适合用 `order_id` 作为 key？

---

## 九、本阶段推荐实验记录

建议建立 `producer-experiments.md`：

```markdown
# Producer 实验记录

## Message Key 实验

- topic:
- partitions:
- key 规则:
- 观察到的 partition:

## acks 实验

- acks:
- 是否等待确认:
- 失败时现象:

## batch/linger/compression 实验

- batch:
- linger:
- compression:
- throughput:
- p95 latency:

## 结论
```

每个参数都要配合现象记录，不要只背定义。

---

## 十、Producer 设计检查表

写任何业务 producer 前，先检查：

- [ ] topic 是否明确。
- [ ] key 是否来自稳定业务字段。
- [ ] payload 是否有 `event_id`。
- [ ] payload 是否有 `event_type`。
- [ ] payload 是否有 `version`。
- [ ] 时间是否使用 UTC。
- [ ] 发送失败是否返回错误。
- [ ] 服务退出前是否 close/flush。

如果这些都没有，producer 只是一个 demo，还不能放进关键业务。

---

## 十一、和后续阶段的关系

第 3 阶段只解决 producer 侧问题，但它会影响后面所有阶段：

```text
key 设计不好 -> consumer 侧顺序和热点问题。
acks 配置过低 -> 消息可能丢。
retry 不了解 -> 重复消息无法解释。
event_id 缺失 -> consumer 无法幂等。
```

所以 producer 阶段不是简单“会发送”，而是为可靠消费和项目落地打基础。

---

## 十二、阶段完成标准

你可以进入第 4 阶段的标志是：

- 能解释 producer 发送路径。
- 能为订单事件选择 key。
- 能说明 `acks=all` 的意义和代价。
- 能说明 producer retry 为什么可能重复。
- 能解释 batch、linger、compression 的取舍。
- 能说明 producer 幂等和业务幂等的区别。

---

## 十三、推荐复盘问题

完成本阶段后，写一段复盘：

```text
如果 order-service 创建订单后要发布 order.created，我会如何构造 event_id？
我会使用什么 message key？
如果 Kafka 暂时不可用，producer 返回错误后业务应该怎么办？
如果 producer retry 导致重复消息，下游如何兜底？
```

这些问题会自然引出后面的 consumer 幂等和 outbox。

---

## 十四、下一阶段预告

第 4 阶段会站在 consumer 视角看同一条消息：

- consumer 如何被分配 partition。
- offset 什么时候提交。
- handler 失败时怎么办。
- rebalance 为什么会让消息重复处理。
- lag 如何反映消费能力。

Producer 决定消息如何进入 Kafka，consumer 决定消息如何安全变成业务结果。

---

## 二十二、阶段案例：订单创建事件

用一个固定案例贯穿本阶段：

```text
业务动作：用户创建订单。
事件名称：order.created。
topic：order.created。
key：order_id。
event_id：evt_order_created_{order_id}。
可靠性：acks=all，失败进入 outbox。
```

后面学习每个 producer 参数，都回到这个案例上问：

```text
这个参数会让 order.created 更可靠、更快，还是更容易排查？
```

---

## 二十三、阶段产物模板

```markdown
# order.created Producer 设计

## Event JSON

## Message Key

## Headers

## Producer 参数

## 发送失败处理

## 退出时 Flush/Close
```

这个模板后续可以直接并入项目设计文档。

---

## 十五、阶段结束小作业

为 `order.created` 写一份 producer 设计：

```text
topic:
message key:
event_id 生成规则:
acks:
retry:
batch/linger:
compression:
发送失败处理:
```

这份设计后面可以直接放进第 8 阶段的生产方案。

---

## 十六、学习产物检查

本阶段结束时，应该留下：

- 一份 `order.created` 事件 JSON 示例。
- 一份 message key 设计说明。
- 一份 acks/retries 配置说明。
- 一份 batch/linger/compression 实验记录。
- 一份 producer 可靠性风险清单。

这些材料会在第 5 阶段 Go producer 封装中变成代码约束。

---

## 十七、面试表达

可以这样描述 producer：

```text
Producer 发送消息时要先决定 topic、key 和 payload。
key 会影响消息进入哪个 partition，从而影响顺序和热点；acks 和 retry 影响可靠性；batch、linger、compression 影响吞吐和延迟。
关键业务里 producer 发送失败不能被忽略，必要时要配合 Outbox。
```

---

## 十八、推荐学习时间安排

| 天数 | 任务 | 产出 |
| --- | --- | --- |
| 第 1 天 | Producer 发送流程 | 流程图 |
| 第 2 天 | Message Key | key 设计表 |
| 第 3 天 | acks/retries | 可靠性对比 |
| 第 4 天 | batch/linger/compression | 性能实验 |
| 第 5 天 | 幂等和顺序性 | 风险清单 |

Producer 参数很多，不要一口气背完。每个参数都要问：它影响可靠性、吞吐、延迟还是顺序？

---

## 十九、Producer 速查表

| 问题 | 优先看 |
| --- | --- |
| 消息可能丢 | acks、错误处理、Outbox |
| 消息可能重复 | retry、幂等、event_id |
| 顺序不稳定 | key、partition、consumer 并发 |
| 吞吐不够 | batch、linger、compression |
| 延迟太高 | linger、batch、broker ack |

---

## 二十、最终检查

进入 Consumer 阶段前，写出一条完整消息的生产设计：

```text
topic + key + event_id + payload + acks + retry + 失败处理
```

能写完整，说明 producer 阶段已经不是停留在 API 层。

---

## 二十一、下一步连接

Producer 阶段结束后，马上进入 Consumer 阶段。发送端做得再可靠，也必须由消费端通过手动提交、幂等和失败处理把消息变成正确的业务结果。

---

## 二十二、本阶段最终交付物

学完 Producer 后，建议留下三份材料：

```text
producer-config.md：记录 acks、retries、linger、batch、compression 的选择理由。
key-design.md：记录每个 topic 使用什么 message key，以及为什么。
send-failure-playbook.md：记录发送失败、超时、重复发送时的处理策略。
```

这三份材料比单纯背参数更重要。真实项目里，Producer 的问题往往不是“不会发消息”，而是没有把顺序、可靠性、吞吐和失败边界提前写清楚。
