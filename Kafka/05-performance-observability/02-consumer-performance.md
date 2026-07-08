# 02 Consumer 性能调优

Consumer 性能瓶颈通常不在 Kafka 本身，而在业务处理、数据库、外部接口和 offset 提交方式。

## 先定位瓶颈

lag 升高时先问：

- producer 写入是否突然变多？
- consumer 是否报错？
- handler 平均耗时是多少？
- 数据库是否慢？
- 是否频繁 rebalance？
- 是否只有某个 partition lag 高？

不要一上来就加 consumer。

## 批量处理

批量处理可以显著提高吞吐。

适合：

- 批量写数据库。
- 批量调用内部服务。
- 日志聚合。

不适合：

- 单条消息业务复杂且要求独立事务。
- 批次中一条失败会影响整个批次的场景。

## 增加 Consumer 实例

增加 consumer 能提升消费能力，但受 partition 数量限制。

如果 topic 只有 3 个 partition，那么同一个 group 中最多只有 3 个 consumer 能同时处理。

多出来的 consumer 会空闲。

## 慢消息处理

某些消息处理特别慢时，可能拖住整个 partition。

解决思路：

- 给外部调用加超时。
- 将慢任务转成异步任务。
- 对失败消息进入 retry topic。
- 按业务 key 拆分热点。
- 增加 partition，但要评估顺序性。

## Offset 提交频率

每条消息都提交 offset，可靠性清晰，但吞吐可能较低。

批量提交吞吐更好，但失败恢复更复杂。

生产中常见做法：

- 关键业务单条或小批量提交。
- 日志类大批量提交。
- 提交失败要记录，允许重复消费。

## Go 侧并发建议

谨慎并发处理同一个 partition。

更稳妥的做法：

- 多 partition 并行。
- partition 内顺序处理。
- handler 内部对非关键 IO 做有限并发。
- worker 数量有上限。
- 所有外部调用有 timeout。

## 本节练习

1. 写一个 consumer，每条消息 sleep 100ms。
2. 发送 10000 条消息，观察 lag。
3. 增加 consumer 数量，观察吞吐变化。
4. 把 topic partition 从 3 调到 6，再观察变化。
5. 总结 partition、consumer、handler 耗时三者关系。

