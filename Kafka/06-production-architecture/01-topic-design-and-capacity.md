# 01 Topic 设计与容量规划

Topic 设计会长期影响系统扩展能力。随手创建 topic，后续很容易在顺序性、吞吐、存储和权限上付出代价。

## Topic 命名

推荐事件式命名：

```text
order.created
order.cancelled
payment.succeeded
inventory.deducted
user.registered
```

命名原则：

- 表达已经发生的事实。
- 用领域对象开头。
- 动作用过去式。
- 不要把 consumer 名称写进 topic。

## Partition 数量

影响因素：

- producer 写入吞吐。
- consumer 并行度。
- 单个 partition 存储大小。
- 未来扩展。
- 顺序性需求。

经验：

- 初学本地实验：3 到 6 个。
- 小型业务 topic：6 到 12 个。
- 高吞吐日志 topic：更多，但要压测和评估。

不要无脑设置非常多 partition。partition 过多会增加集群管理成本。

## Replication Factor

生产环境常见设置为 3。

本地单节点只能设置 1。

关键 topic 应该：

- replication factor = 3。
- min.insync.replicas 合理设置。
- producer 使用 `acks=all`。

## Retention

设计 retention 要考虑：

- 消费者最长可能宕机多久。
- 是否需要历史重放。
- 合规和审计要求。
- 磁盘成本。

订单、支付等关键事件通常需要较长保留时间。

日志类 topic 可以根据分析窗口设置。

## 容量估算

简化公式：

```text
每日数据量 = 平均消息大小 * 每日消息数
总存储 = 每日数据量 * 保留天数 * 副本数
```

示例：

```text
平均消息大小：2KB
每日消息数：1000 万
保留时间：7 天
副本数：3

每日原始数据：2KB * 1000 万 = 约 20GB
总存储：20GB * 7 * 3 = 约 420GB
```

还要预留：

- 索引文件。
- 日志开销。
- 磁盘水位。
- 流量增长。

## 本节练习

1. 为订单、支付、库存、通知设计 topic。
2. 为每个 topic 选择 partition 数量。
3. 估算 `order.created` 每日数据量。
4. 解释为什么关键 topic 不应该只保留 1 小时。

