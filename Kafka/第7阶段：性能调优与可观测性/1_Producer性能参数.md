# 1. Producer 性能参数

本节目标：理解 producer 侧影响吞吐和延迟的关键参数，知道如何根据业务目标选择 batch、linger、compression、acks 等配置。

Kafka 性能调优不是背参数，而是做取舍：吞吐、延迟、可靠性、CPU、网络、磁盘之间不可能全部同时最优。

---

## 一、先明确业务目标

调参前先问：

```text
这个 topic 是关键业务事件，还是日志？
更关注吞吐，还是低延迟？
消息能不能丢？
平均消息大小是多少？
峰值 QPS 是多少？
```

订单事件和用户行为日志的配置不应该完全一样。

---

## 二、batch.size

batch 控制 producer 批量发送。

增大 batch 可能：

- 提升吞吐。
- 提升压缩率。
- 增加内存占用。
- 增加等待时间。

如果消息量很低，batch 再大也不一定凑得满。

---

## 三、linger.ms

`linger.ms` 控制 producer 为凑 batch 最多等待多久。

例如：

```text
linger.ms=20
```

表示最多等 20ms。

优点：

- 更容易凑批。
- 提高吞吐。

缺点：

- 增加端到端延迟。

---

## 四、compression.type

常见：

```text
none
lz4
zstd
snappy
gzip
```

压缩优点：

- 降低网络流量。
- 降低磁盘占用。

代价：

- 增加 producer CPU。
- 增加 consumer 解压 CPU。

大 JSON 消息通常适合压缩。

---

## 五、acks 与性能

`acks=all` 更可靠，但延迟可能更高。

`acks=1` 延迟可能更低，但可靠性下降。

`acks=0` 延迟低但丢失风险高。

关键业务优先可靠性：

```text
acks=all
```

日志类可以结合业务接受度选择。

---

## 六、推荐配置方向

订单事件：

```text
acks=all
enable.idempotence=true
linger.ms=5~20
compression=lz4 或 zstd
```

行为日志：

```text
batch.size 较大
linger.ms=20~100
compression=lz4 或 zstd
acks=1 或 all
```

低延迟通知：

```text
linger.ms=0~5
batch.size 适中
compression 视消息大小决定
```

---

## 七、Go 侧注意事项

- producer 要复用。
- Publish 要支持 context。
- 失败要记录。
- 异步发送要处理 delivery report。
- 退出前 close/flush。

---

## 八、压测指标

记录：

- 每秒消息数。
- 每秒字节数。
- 平均延迟。
- P95 延迟。
- P99 延迟。
- 错误率。
- producer CPU。

不要只看平均值。

---

## 九、常见误区

- batch 越大越好。
- linger 越大越好。
- 压缩一定更快。
- 所有业务共用一套配置。
- 只测本地单节点就推导生产结果。

---

## 十、本节练习

1. 为订单事件设计 producer 参数。
2. 为行为日志设计 producer 参数。
3. 解释 batch 和 linger 的关系。
4. 解释压缩为什么会增加 CPU。
5. 设计一个 producer 压测表格。

---

## 十一、本节小结

- Producer 调优是吞吐、延迟、可靠性的取舍。
- batch 和 linger 影响批量。
- compression 降低网络和磁盘，增加 CPU。
- acks 影响可靠性和延迟。
- 不同业务应该有不同配置。

---

## 十二、最小压测程序思路

学习阶段可以写一个简单 Go 程序，连续发送 N 条消息，记录总耗时。

伪代码：

```go
func main() {
    total := 100000
    started := time.Now()

    for i := 0; i < total; i++ {
        event := buildOrderCreated(i)
        payload, _ := json.Marshal(event)

        err := producer.Publish(ctx, "perf.order.created", kafka.Message{
            Key:   []byte(event.Data.OrderID),
            Value: payload,
            Headers: map[string]string{
                "event_id": event.EventID,
            },
        })
        if err != nil {
            failed++
        }
    }

    cost := time.Since(started)
    fmt.Printf("total=%d failed=%d cost=%s throughput=%.2f msg/s\n",
        total, failed, cost, float64(total)/cost.Seconds())
}
```

这个程序很粗糙，但足够让你观察不同参数下的变化。

---

## 十三、压测变量设计

不要一次改很多参数。

建议一轮只改一个变量：

| 场景 | batch | linger | compression | acks |
| --- | --- | --- | --- | --- |
| baseline | 默认 | 默认 | none | all |
| 压缩 lz4 | 默认 | 默认 | lz4 | all |
| 压缩 zstd | 默认 | 默认 | zstd | all |
| 增加 linger | 默认 | 20ms | lz4 | all |
| 日志吞吐 | 较大 | 50ms | lz4 | 1 |

记录：

- 吞吐。
- 平均延迟。
- P95。
- P99。
- 错误数。
- CPU。

---

## 十四、P95 和 P99 为什么重要

平均值可能掩盖尾部延迟。

例如：

```text
平均 10ms
P95 30ms
P99 900ms
```

这说明大多数消息很快，但少量消息非常慢。

用户或下游服务常常感知的是尾部延迟，而不是平均延迟。

---

## 十五、订单事件配置示例

订单事件更看重可靠性：

```text
acks=all
enable.idempotence=true
compression=lz4
linger.ms=5~20
delivery.timeout.ms 合理设置
```

为什么不是把 linger 设置很大？

因为创建订单后，下游库存、通知可能希望尽快收到事件。过大的 linger 会增加端到端延迟。

---

## 十六、行为日志配置示例

行为日志更看重吞吐：

```text
acks=1 或 all
compression=zstd 或 lz4
linger.ms=50~100
batch.size 较大
```

行为日志通常可以接受几十毫秒甚至更高延迟，用更大的 batch 换吞吐。

---

## 十七、调参报告模板

```markdown
# Producer 调参报告

## 目标

## 环境

## Topic 配置

## 消息大小

## 参数组合

## 结果

| 场景 | 吞吐 | Avg | P95 | P99 | 错误数 |
| --- | --- | --- | --- | --- | --- |

## 观察

## 结论

## 下一步
```

---

## 十八、面试回答思路

如果被问“Kafka producer 如何提升吞吐”，不要只背参数。

可以这样回答：

```text
先确认业务目标。如果是日志类高吞吐场景，可以增大 batch，适当增加 linger，开启 lz4/zstd 压缩，减少单条请求开销。但如果是订单事件，要优先保证可靠性，通常 acks=all、开启幂等，再在可接受延迟内调 batch 和 linger。调优必须通过压测看 P95/P99、错误率和 CPU。
```

---

## 十九、实验组合建议

不要一次改太多参数。可以按下面组合测试：

| 场景 | batch | linger | compression |
| --- | --- | --- | --- |
| baseline | 默认 | 默认 | none |
| 小延迟 | 小 | 0-5ms | none |
| 高吞吐 | 大 | 10-50ms | lz4 |
| 大消息 | 大 | 10ms | zstd |

每组记录：

```text
messages/s
avg latency
p95 latency
p99 latency
producer CPU
broker network in
```

---

## 二十、订单事件推荐起点

对 `order.created` 这类关键业务，可以从保守配置开始：

```text
acks=all
enable.idempotence=true
linger=5ms
compression=lz4
```

然后通过压测确认延迟是否满足业务要求。不要为了吞吐先牺牲可靠性。

---

## 二十一、参数决策表

| 业务类型 | 推荐方向 | 说明 |
| --- | --- | --- |
| 订单事件 | 可靠性优先 | `acks=all`，适度 batch |
| 用户行为日志 | 吞吐优先 | 更大 batch，允许更高 linger |
| 告警事件 | 低延迟优先 | 小 linger，快速发送 |
| 大 JSON 事件 | 压缩优先 | 观察 CPU 和网络收益 |

调参前先确认业务类型。不同业务共用一套 producer 参数，往往不是最优解。

---

## 二十二、最终验收

完成本节后，要能给出两套参数：

```text
order.created 参数：可靠优先。
user.behavior 参数：吞吐优先。
```

并解释每个参数为什么这样选。

---

## 二十三、最终练习

写一段压测结论：

```text
开启 lz4 后网络流量下降，但 producer CPU 上升。
P99 延迟仍在业务可接受范围内，因此保留压缩。
```

结论要同时看收益和代价。

---

## 二十一、Producer 指标对照表

调 Producer 参数时，至少同时看：

| 指标 | 说明 | 可能动作 |
| --- | --- | --- |
| send rate | 发送速率 | 判断吞吐是否提升 |
| request latency | broker 请求延迟 | 判断是否被 broker 或网络拖慢 |
| record error rate | 发送错误率 | 判断可靠性风险 |
| batch size | 批次大小 | 判断 batch 是否真正聚合 |
| compression ratio | 压缩比 | 判断压缩收益 |

如果 batch 参数变了，但实际 batch size 没变化，说明业务流量或 linger 设置可能不足以形成批量。
