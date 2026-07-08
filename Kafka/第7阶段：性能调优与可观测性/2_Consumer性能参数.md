# 2. Consumer 性能参数

本节目标：理解 consumer 侧影响吞吐和稳定性的关键因素，知道什么时候该优化 handler、增加 consumer、增加 partition 或批量处理。

Consumer 性能问题经常不在 Kafka，而在业务处理：数据库慢、外部接口慢、锁竞争、坏消息重试。调参前要先定位瓶颈。

---

## 一、Consumer 性能由什么决定

主要因素：

- topic partition 数。
- consumer 实例数。
- 每条消息 handler 耗时。
- offset 提交方式。
- 下游数据库和接口性能。
- 是否频繁 rebalance。

---

## 二、partition 与 consumer 数

同一个 group：

```text
最大有效 consumer 数 <= partition 数
```

如果 partition=3，consumer=10，只有 3 个会真正消费。

扩容 consumer 前先看 partition。

---

## 三、handler 耗时

Consumer 吞吐近似受 handler 耗时影响。

如果单条处理 100ms，一个 partition 顺序处理理论上约：

```text
10 条/秒
```

如果 6 个 partition 并行：

```text
约 60 条/秒
```

实际还受数据库、网络、提交 offset 影响。

---

## 四、批量处理

批量处理适合：

- 日志入库。
- 行为数据。
- 批量写分析系统。

不适合：

- 单条业务事务复杂。
- 中间失败难处理。
- 强幂等要求但未设计好。

---

## 五、提交 offset 成本

每条都提交：

- 边界清晰。
- 吞吐较低。

批量提交：

- 吞吐更好。
- 中间失败复杂。

关键业务先用单条或小批量，稳定后再优化。

---

## 六、慢消息处理

慢消息会拖住 partition。

处理方式：

- 外部调用加 timeout。
- 可重试错误进入 retry topic。
- 不可重试错误进入 DLQ。
- 慢任务拆出去。
- 优化数据库。

---

## 七、Go 并发注意事项

不要轻易对同一个 partition 内消息无序并发处理。

问题：

- 顺序被破坏。
- offset 提交边界复杂。
- 前一条失败后一条成功，不好提交。

更安全：

- 多 partition 并行。
- partition 内顺序。
- handler 内部做有限并发。

---

## 八、常见参数直觉

不同客户端名称可能不同，但概念类似：

- fetch min bytes。
- fetch max bytes。
- max poll records。
- session timeout。
- max poll interval。

核心问题是：

```text
一次拉多少？
多久没响应算异常？
处理多久会被认为不健康？
```

---

## 九、优化顺序

建议：

1. 看 lag 是全局还是单 partition。
2. 看 handler P95/P99。
3. 看错误率。
4. 看下游数据库。
5. 看 rebalance。
6. 再考虑加 consumer 或 partition。

---

## 十、本节练习

1. 估算单条处理 50ms 时单 partition 吞吐。
2. 解释为什么 consumer 数超过 partition 数无效。
3. 设计慢消息处理方案。
4. 思考批量提交 offset 的风险。
5. 设计 consumer 处理耗时指标。

---

## 十一、本节小结

- Consumer 性能常受业务 handler 影响。
- partition 限制 group 并行度。
- 加 consumer 前先看 partition。
- 批量处理提升吞吐但复杂。
- 同 partition 内无序并发要谨慎。
- 慢消息应通过 timeout、retry、DLQ 处理。

---

## 十二、用数字建立直觉

假设一个 topic 有 3 个 partition。

如果每条消息 handler 耗时 100ms，且每个 partition 顺序处理：

```text
单 partition 约 10 条/秒
3 partition 约 30 条/秒
```

如果 producer 每秒写入 100 条：

```text
consumer 处理能力 30 条/秒
写入速度 100 条/秒
lag 每秒大约增加 70 条
```

这就是 lag 持续增长的根本原因。

---

## 十三、实验：模拟慢 Handler

你可以在 Go consumer handler 中临时加入：

```go
time.Sleep(100 * time.Millisecond)
```

然后发送 1000 条消息。

观察：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group perf-inventory-service
```

记录 lag 变化。

实验变量：

| handler sleep | partition | consumer | 预期 |
| --- | --- | --- | --- |
| 10ms | 3 | 3 | lag 下降较快 |
| 50ms | 3 | 3 | 下降变慢 |
| 100ms | 3 | 3 | 下降更慢 |
| 100ms | 3 | 6 | 不会明显超过 3 consumer |
| 100ms | 6 | 6 | 并行能力提升 |

---

## 十四、批量写数据库示例

日志类 consumer 可以把多条消息聚合后批量写入：

```go
batch := make([]Event, 0, 100)

for _, msg := range messages {
    event := decode(msg)
    batch = append(batch, event)
}

err := repo.BatchInsert(ctx, batch)
if err != nil {
    return err
}
```

但订单、库存这种关键业务要谨慎，因为批次中某一条失败时，offset 提交边界会复杂。

---

## 十五、外部调用 Timeout

Consumer handler 中所有外部调用都应该有 timeout。

```go
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()

err := paymentClient.Call(ctx, req)
```

没有 timeout 的后果：

- goroutine 卡住。
- poll 间隔变长。
- rebalance。
- lag 增长。

---

## 十六、优化决策表

| 现象 | 优先处理 |
| --- | --- |
| 所有 partition lag 高 | 看写入量、handler 耗时、下游 |
| 单 partition lag 高 | 查热点 key、坏消息 |
| consumer 数小于 partition | 可以尝试扩 consumer |
| consumer 数大于 partition | 扩 consumer 无效 |
| P99 很高 | 查慢消息、慢 SQL、外部接口 |
| rebalance 频繁 | 查 handler 超时、进程重启、心跳 |

---

## 十七、Go 后端实践建议

关键业务 consumer：

- 单条或小批量处理。
- 手动提交 offset。
- handler 有 timeout。
- 业务幂等。
- retry/DLQ。
- 记录 P95/P99。

日志类 consumer：

- 批量处理。
- 批量提交。
- 压缩。
- 更关注吞吐。

---

## 十八、练习补充

1. 假设 handler 20ms、partition 6，估算理论吞吐。
2. 如果写入速度大于消费速度，画出 lag 曲线。
3. 设计一个实验验证 consumer 数超过 partition 后无效。
4. 写出慢 SQL 导致 lag 增长的排查流程。

---

## 十九、吞吐估算公式

粗略估算：

```text
单 consumer 吞吐 ≈ 1 / handler 平均耗时
总吞吐 ≈ min(consumer 数, partition 数) * 单 consumer 吞吐
```

如果 handler 平均 `20ms`：

```text
单 consumer ≈ 50 msg/s
6 partition + 6 consumer ≈ 300 msg/s
```

真实结果还会受数据库、网络、批量提交、消息大小影响。

---

## 二十、参数调优顺序

先观察，再调参：

1. 看 lag 是否持续增长。
2. 看 handler 耗时。
3. 看错误率。
4. 看 partition 是否均匀。
5. 再调整 consumer 数、批量大小和数据库 SQL。

---

## 二十一、慢 Handler 示例

假设库存扣减 SQL 平均耗时从 `20ms` 变成 `200ms`：

```text
单 consumer 吞吐从 50 msg/s 降到 5 msg/s。
6 个 partition + 6 个 consumer 的理论吞吐从 300 msg/s 降到 30 msg/s。
```

这时 lag 增长不是 Kafka 慢，而是业务处理慢。

---

## 二十二、调优动作表

| 现象 | 优先动作 |
| --- | --- |
| 所有 partition lag 均匀增长 | 增加 consumer 或优化 handler |
| 单 partition lag 高 | 排查热点 key |
| handler P99 高 | 优化 SQL 或外部依赖 |
| commit 很慢 | 检查网络和 broker 状态 |
| rebalance 频繁 | 检查 session、heartbeat、处理耗时 |

Consumer 性能调优一定要结合业务耗时看。

---

## 二十三、最终检查

调优前先写出当前瓶颈假设：

```text
我认为 lag 增长是因为 handler P99 过高，而不是 Kafka broker 吞吐不足。
```

然后用指标验证这个假设。

---

## 二十四、实验设计示例

固定 topic 为 6 个 partition，分别测试：

| consumer 数 | handler 耗时 | 预期 |
| --- | --- | --- |
| 3 | 20ms | 吞吐受 consumer 数限制 |
| 6 | 20ms | 接近 partition 并行上限 |
| 12 | 20ms | 超过 partition 的实例不会继续提升 |
| 6 | 200ms | handler 成为瓶颈 |

实验时只改一个变量，结果才有解释价值。

---

## 二十五、报告结论模板

```text
本次 lag 增长主要由 handler P99 升高导致。
当前 partition 数为 6，consumer 数为 6，继续增加 consumer 收益有限。
下一步优先优化库存扣减 SQL，并复测 handler P99。
```

---

## 二十六、最终验收

完成本节后，应该能根据 handler 耗时估算 consumer 吞吐，并用实际 lag 曲线验证估算是否接近。

估算不要求绝对准确，但要能指导扩容或优化方向。

---

## 二十一、Consumer 吞吐估算例子

假设：

```text
单条 handler 平均耗时：20ms
单个 consumer 串行处理：约 50 条/秒
topic 有 6 个 partition
部署 3 个 consumer 实例
理论总处理能力：约 150 条/秒
```

如果 producer 写入速度长期是 300 条/秒，那么 lag 持续增长是正常结果。

这时要么优化 handler，要么增加并行度，要么增加 partition 后再扩 consumer，而不是只改 `max.poll.records`。
