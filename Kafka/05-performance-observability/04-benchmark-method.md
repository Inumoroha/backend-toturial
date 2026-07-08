# 04 压测方法

本节目标：学会设计 Kafka 压测，而不是随便跑一个数字。

## 压测前先定义问题

不要问“Kafka 能跑多快”。要问：

- 在我们的消息大小下，producer 吞吐是多少？
- 开启压缩后延迟变化多少？
- consumer 每秒能处理多少订单事件？
- 数据库变慢时 lag 会怎样增长？
- 增加 partition 是否有效？

## 记录环境

每次压测记录：

- Kafka 版本。
- broker 数量。
- topic partition 数。
- replication factor。
- 消息大小。
- producer 参数。
- consumer 参数。
- Go 程序版本。
- 机器 CPU、内存、磁盘。

没有环境信息的压测结果没有比较价值。

## Producer 压测

测试维度：

- 单条消息大小。
- batch.size。
- linger.ms。
- compression.type。
- acks。
- producer 实例数。

记录结果：

- 每秒消息数。
- 每秒字节数。
- 平均延迟。
- P95/P99 延迟。
- 失败率。

## Consumer 压测

测试维度：

- partition 数。
- consumer 数。
- handler 耗时。
- 单条提交 vs 批量提交。
- 批量写数据库。
- 下游超时。

记录结果：

- 每秒处理数。
- lag 变化曲线。
- handler P95/P99。
- 错误率。
- rebalance 次数。

## 压测结论模板

```markdown
# 压测结论

## 问题

## 环境

## 参数

## 结果

## 观察

## 结论

## 下一步
```

## 常见误区

- 只测 producer，不测 consumer。
- 只看平均延迟，不看 P95/P99。
- 本地单节点结果直接套到生产。
- 不记录消息大小。
- 不记录错误率。
- 压测时下游数据库是假实现，结论却用于真实业务。

## 本节练习

1. 设计一个 100 万条消息的 producer 压测。
2. 设计一个 consumer 慢处理压测。
3. 对比 3 partition 和 6 partition 的消费能力。
4. 写一份压测结论，不超过一页。

