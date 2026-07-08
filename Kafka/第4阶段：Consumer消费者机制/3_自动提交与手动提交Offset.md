# 3. 自动提交与手动提交 Offset

本节目标：理解自动提交和手动提交 offset 的区别，掌握 Go 后端关键业务为什么通常选择手动提交。

Offset 提交时机是 Kafka consumer 可靠性的核心。提交太早可能丢消息，提交太晚可能重复消费。关键业务的正确策略通常是：业务处理成功后再提交。

---

## 一、提交 Offset 的含义

提交 offset 是告诉 Kafka：

```text
这个 consumer group 已经处理到这里了
```

Kafka 会记录：

```text
group + topic + partition + offset
```

下次 consumer 重启，会从提交位置继续消费。

---

## 二、自动提交是什么

自动提交是客户端定期提交 offset。

例如：

```text
每 5 秒提交一次当前 poll 到的位置
```

优点：

- 配置简单。
- 代码少。
- 适合学习或非关键日志。

缺点：

- 业务可能还没处理成功，offset 已经提交。
- 失败时不容易精细控制。
- 不适合订单、支付、库存。

---

## 三、自动提交的风险

场景：

```text
consumer 拉到 order.created
自动提交 offset
开始扣库存
数据库超时
consumer 崩溃
```

重启后：

```text
Kafka 认为这条消息已处理
consumer 从后面继续
库存没有扣减
```

这就是消息处理机会丢失。

---

## 四、手动提交是什么

手动提交由应用代码决定什么时候提交 offset。

推荐流程：

```text
拉取消息
解析消息
业务处理
处理成功
提交 offset
```

如果处理失败：

```text
不提交，或写 retry/DLQ 成功后提交
```

---

## 五、手动提交的代价

手动提交更可靠，但更复杂。

你必须处理：

- handler 成功但 commit 失败。
- handler 失败是否重试。
- 不可重试错误是否进 DLQ。
- consumer 退出时是否还有消息未处理。
- 批量处理时某条失败怎么办。

所以手动提交不是“多写一行 commit”，而是一套错误处理设计。

---

## 六、提交 Offset 的决策表

| 场景 | 是否提交 offset |
| --- | --- |
| 业务处理成功 | 提交 |
| 重复消息已识别且无需再处理 | 提交 |
| 可重试错误，本地短重试中 | 不提交 |
| 可重试错误，已写 retry topic | 提交 |
| 不可重试错误，已写 DLQ | 提交 |
| 写 retry 失败 | 不提交 |
| 写 DLQ 失败 | 不提交 |
| 进程退出，消息未处理完 | 不提交 |

核心原则：

```text
只有消息已经被成功处理，或成功转移到可靠后续链路，才能提交 offset
```

---

## 七、Go Consumer 伪代码

```go
for {
    msg := consumer.Fetch(ctx)

    err := handler.Handle(ctx, msg)
    if err == nil {
        consumer.Commit(ctx, msg)
        continue
    }

    if IsRetryable(err) {
        if sendRetry(ctx, msg, err) == nil {
            consumer.Commit(ctx, msg)
        }
        continue
    }

    if sendDLQ(ctx, msg, err) == nil {
        consumer.Commit(ctx, msg)
    }
}
```

注意：

```text
写 retry 或 DLQ 成功后，才提交原消息 offset
```

---

## 八、Commit 失败怎么办

业务成功，但 commit offset 失败：

```text
消息可能会被重复消费
```

所以 consumer handler 必须幂等。

例如库存扣减：

```text
processed_events(event_id) 唯一约束
```

重复消费时：

```text
发现 event_id 已处理
直接返回成功
再次提交 offset
```

---

## 九、批量提交

批量提交可以提升吞吐，但失败边界更复杂。

例如一批消息：

```text
offset 10 成功
offset 11 成功
offset 12 失败
offset 13 成功
```

此时不能简单提交到 14，因为 offset 12 失败了。

否则重启后会跳过 offset 12。

关键业务初学阶段建议先单条提交，理解清楚后再优化。

---

## 十、命令行能否演示手动提交

命令行 consumer 默认行为和真实客户端细节不同，不适合完整演示手动提交。

手动提交更适合在 Go 代码中演示。

但你可以通过 group lag 观察 offset 变化：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group your-group
```

---

## 十一、常见误区

### 1. 自动提交永远不可靠

不是。

日志类、可丢场景可以用自动提交。

但关键业务不建议。

### 2. 手动提交就不会重复

不是。

commit 失败、rebalance、进程崩溃都可能导致重复。

### 3. 失败时永远不提交

不对。

不可重试错误如果不提交，会卡住 partition。应该写 DLQ 后提交。

### 4. 批量提交很简单

不一定。

中间失败会让提交边界复杂。

---

## 十二、本节练习

1. 写出自动提交可能丢消息的流程。
2. 写出手动提交的推荐流程。
3. 解释业务成功但 commit 失败会发生什么。
4. 设计 JSON 解析失败时的 offset 提交策略。
5. 思考批量处理时 offset 12 失败，offset 13 成功，应该提交到哪里。

---

## 十三、本节小结

- offset 提交表示 group 的消费进度。
- 自动提交简单，但可能业务未成功就提交。
- 手动提交更适合关键业务。
- 业务成功后提交 offset。
- 写 retry/DLQ 成功后可以提交原 offset。
- commit 失败会导致重复消费，所以业务必须幂等。
- 批量提交要谨慎处理中间失败。

---

## 十四、Go 手动提交模板

伪代码：

```go
for {
    msg, err := reader.FetchMessage(ctx)
    if err != nil {
        return err
    }

    if err := handler.Handle(ctx, msg); err != nil {
        if handled := handleFailure(ctx, msg, err); !handled {
            continue
        }
    }

    if err := reader.CommitMessages(ctx, msg); err != nil {
        logger.Error("commit offset failed",
            "topic", msg.Topic,
            "partition", msg.Partition,
            "offset", msg.Offset,
            "error", err,
        )
    }
}
```

提交失败不能简单退出并假装没事。你要知道它可能导致重复消费，所以 handler 必须幂等。

---

## 十五、失败分类下的提交策略

| 处理结果 | 下一步 | 是否提交 offset |
| --- | --- | --- |
| 业务成功 | 无 | 是 |
| 可重试错误 | 写 retry topic | retry 写成功后提交 |
| 不可重试错误 | 写 DLQ | DLQ 写成功后提交 |
| retry 写失败 | 保留原消息 | 否 |
| DLQ 写失败 | 保留原消息 | 否 |

一句话：

```text
消息没有成功处理，也没有成功转移到可靠后续链路，就不能提交 offset。
```

---

## 十六、批量提交的安全边界

如果批量处理 offsets：

```text
10 成功
11 成功
12 失败
13 成功
14 成功
```

不能提交到 `15`，因为 `12` 还没成功。最多只能提交到 `12` 之前的位置。

这就是批量并发消费复杂的地方：你需要维护连续成功区间，而不是看到某条成功就提交到最后。

---

## 十七、单条处理推荐流程

初学和关键业务建议先用单条处理：

```text
fetch one message
handle business
if success:
  commit this offset
else:
  retry or dlq
```

单条处理吞吐可能不如批量，但语义清晰，适合订单、库存、支付这类业务。

---

## 十八、面试表达

```text
自动提交 offset 简单，但可能在业务处理成功前提交进度。
关键业务我会选择手动提交，业务成功后提交 offset；如果失败消息成功写入 retry 或 DLQ，也可以提交原 offset。
commit 失败可能导致重复消费，所以业务必须幂等。
```

---

## 十九、Offset 提交排查

如果出现“消息明明处理失败却不再消费”，优先检查：

```text
是否开启了自动提交。
是否在 handler 前提交了 offset。
handler 失败后是否仍然返回 nil。
retry/DLQ 写失败后是否仍然提交。
```

offset 问题一旦发生，排查时要同时看业务日志和 consumer group offset。

---

## 二十、实验建议

故意让 handler 在处理后、提交前退出进程。

预期：

```text
重启后同一条消息会再次被消费。
如果业务幂等，结果不会重复产生副作用。
```

---

## 二十、手动提交的代码边界

手动提交 offset 时，建议在代码里坚持一个原则：

```text
handler 只负责业务处理。
consumer 框架负责提交 offset。
```

也就是说，业务函数返回成功或失败，由外层 consumer 决定是否提交。

这样做有两个好处：

- 业务代码不需要知道 Kafka 细节。
- 提交策略可以统一处理 retry、DLQ 和日志。

如果每个 handler 自己提交 offset，后续排查重复消费会非常困难。

---

## 二十一、提交策略选择题

遇到下面场景时，建议选择手动提交：

```text
处理逻辑会写数据库。
处理失败需要进入 retry 或 DLQ。
业务要求失败后不能静默跳过。
需要确保成功处理后再推进 offset。
```

自动提交适合学习和简单 demo。真实 Go 后端服务里，只要消息会产生业务副作用，手动提交通常更容易把语义说清楚。
