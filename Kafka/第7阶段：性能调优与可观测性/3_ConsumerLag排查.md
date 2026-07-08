# 3. Consumer Lag 排查

本节目标：建立 consumer lag 排查流程，遇到 lag 持续升高时能按步骤定位，而不是只会加机器。

---

## 一、确认 Lag

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

记录：

- topic。
- partition。
- current offset。
- log end offset。
- lag。
- consumer id。

---

## 二、判断形态

全 partition lag 高：

- 写入流量突增。
- consumer 总体处理能力不足。
- 下游整体变慢。

单 partition lag 高：

- 热点 key。
- 坏消息。
- 某个 consumer 异常。

---

## 三、排查写入流量

先问：

```text
producer 是否突然变多？
是否有历史重放？
是否有定时任务批量写入？
```

如果写入暴涨，lag 增长可能是正常排队。

---

## 四、排查错误率

看 consumer 日志：

```text
decode failed
database timeout
retry publish failed
dlq publish failed
commit failed
```

错误率升高会降低有效消费吞吐。

---

## 五、排查 handler 耗时

关注：

- avg。
- P95。
- P99。
- max。

P99 高说明少数慢请求拖住 partition。

---

## 六、排查下游依赖

常见瓶颈：

- 数据库慢查询。
- 数据库连接池满。
- Redis 超时。
- HTTP 下游慢。
- 第三方接口限流。

所有外部调用必须有 timeout。

---

## 七、排查 rebalance

频繁 rebalance 日志：

```text
join group
sync group
heartbeat failed
session timeout
max poll interval exceeded
```

可能原因：

- handler 太慢。
- consumer 频繁重启。
- CPU/GC 问题。
- 网络抖动。

---

## 八、止血方式

- 临时扩 consumer，但不能超过 partition 上限。
- 暂停非关键 producer。
- 将坏消息转 DLQ。
- 降级慢下游。
- 临时提高数据库资源，但要谨慎。

---

## 九、根因修复

- 优化 handler。
- 批量处理。
- 增加 partition。
- 拆分热点 key。
- 完善 retry/DLQ。
- 增加监控和告警。

---

## 十、本节练习

1. 写出 lag 排查清单。
2. 解释单 partition lag 高的三种原因。
3. 解释为什么 lag 高不一定是 Kafka broker 问题。
4. 设计订单 consumer lag 告警。

---

## 十一、本节小结

- lag 要看 partition 维度。
- lag 高先看写入、错误、耗时、下游、rebalance。
- 加 consumer 不一定有效。
- 单 partition lag 高常见于热点 key 或坏消息。
- 可观测性是排查 lag 的基础。

---

## 十二、完整排查案例：库存服务 Lag 升高

现象：

```text
inventory-service lag 从 0 增长到 50000，并持续 20 分钟不下降。
```

第一步查看 group：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

假设输出显示：

```text
PARTITION 0 LAG 100
PARTITION 1 LAG 49800
PARTITION 2 LAG 100
```

结论：

```text
不是整体消费能力不足，而是 partition 1 异常。
```

下一步查 partition 1 对应日志：

```text
topic=order.created partition=1 offset=203391 key=sku_hot_001 error=database timeout
```

可能原因：

- 热点 key。
- 这条消息一直失败。
- 对应 consumer 实例访问数据库异常。

---

## 十三、完整排查案例：所有 Partition Lag 都高

假设输出：

```text
partition 0 lag 20000
partition 1 lag 22000
partition 2 lag 21000
```

这更像整体问题。

排查顺序：

1. producer 写入是否暴涨。
2. consumer 实例是否减少。
3. handler P95/P99 是否升高。
4. 数据库连接池是否满。
5. 是否有频繁 rebalance。

如果发现：

```text
handler P99 从 100ms 升到 5s
```

就不要先怪 Kafka。应该先查下游数据库或外部接口。

---

## 十四、排查记录模板

```markdown
# Consumer Lag 排查记录

## 发现时间

## 影响范围

## Group

## Topic

## Partition Lag

| partition | current offset | log end offset | lag |
| --- | --- | --- | --- |

## Producer 写入变化

## Consumer 错误率

## Handler P95/P99

## Rebalance 情况

## 下游依赖状态

## 根因

## 临时处理

## 长期修复
```

---

## 十五、不要直接 Reset Offset

Kafka 支持 reset offset，但这是危险操作。

如果你直接把 offset 跳到最新：

```text
积压消息会被跳过
```

对日志类可能可以接受。

对订单、支付、库存类通常不能接受。

Reset offset 前必须确认：

- 消息是否已备份。
- 是否可以通过 DLQ 或补偿恢复。
- 业务是否接受跳过。
- 是否有审批。

---

## 十六、单 Partition Lag 高的处理手段

如果是坏消息：

```text
定位 offset
分析 payload
修复代码或数据
必要时写 DLQ 后提交
```

如果是热点 key：

```text
评估是否必须保持该 key 顺序
拆分 key
拆分 topic
增加业务分片
```

如果是 consumer 实例异常：

```text
重启实例
检查 CPU/内存/网络
检查数据库连接
```

---

## 十七、面试回答思路

如果被问“Kafka consumer lag 太高怎么排查”，可以按这条线回答：

```text
先用 kafka-consumer-groups 查看 lag 是全部 partition 高还是单 partition 高。
如果全部高，看 producer 写入是否暴涨、consumer 是否减少、handler 是否变慢、下游依赖是否异常、是否 rebalance。
如果单 partition 高，重点查热点 key、坏消息或对应 consumer 实例异常。
解决上，能扩 consumer 的前提是 consumer 数小于 partition 数；坏消息进 DLQ，慢 handler 优化或拆分，热点 key 需要重新设计分区策略。
```

---

## 十八、排查记录模板

每次 lag 告警后，建议记录：

```text
告警时间：
group：
topic：
总 lag：
最大 partition lag：
producer 写入速率：
consumer 实例数：
handler P99：
错误率：
DLQ 新增：
最终原因：
处理动作：
```

这份记录能帮助团队判断问题是否反复出现。

---

## 十九、常见错误动作

### 1. 直接重置 offset

重置 offset 可能导致消息丢失或大量重复处理。除非你非常清楚业务后果，否则不要把它当成常规排障手段。

### 2. 只重启 consumer

重启可能短期恢复，但如果根因是慢 SQL、热点 key 或坏消息，问题很快会回来。

### 3. 只看总 lag

总 lag 高不代表所有 partition 都高。单 partition lag 高通常更像热点或坏消息。

---

## 二十、Go 后端落地

Go consumer 日志里至少要有：

```text
topic
partition
offset
key
event_id
handler
duration_ms
error
```

没有这些字段，lag 排查会非常慢。

---

## 二十一、最终验收

完成本节后，应该能根据一段 group describe 输出判断：

```text
是所有 partition 都落后，还是单个 partition 落后。
是消费能力问题，还是热点 key 或坏消息问题。
```

只有能做出判断，lag 指标才真正有用。

---

## 二十二、最终练习

给自己构造一个场景：

```text
partition 0 lag=0
partition 1 lag=20000
partition 2 lag=0
```

写出至少三种可能原因和对应排查动作。

---

## 二十、排查动作要能落地

针对上面的不均衡 lag，可以给出更具体的动作：

```text
检查 message key 是否集中到少数值。
查看 partition 1 的消费实例是否有错误日志。
比较各 partition 的写入速率。
检查 handler 是否因为某类订单卡住。
确认是否刚发生过 rebalance。
```

如果最后发现是 key 热点，单纯增加 consumer 实例不会解决问题，因为热点 partition 仍然只能由一个组内成员消费。
