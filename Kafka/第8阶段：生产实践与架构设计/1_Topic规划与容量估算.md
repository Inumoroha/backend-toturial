# 1. Topic 规划与容量估算

本节目标：学会为生产业务设计 topic、partition、replication factor、retention，并做基础容量估算。

---

## 一、Topic 命名

推荐：

```text
order.created
order.cancelled
payment.succeeded
inventory.deducted
```

原则：

- 表达已经发生的事实。
- 使用领域对象加过去式动作。
- 不把 consumer 名字写进 topic。

---

## 二、Partition 数量

影响因素：

- 写入 QPS。
- consumer 并行度。
- broker 数量。
- 顺序性需求。
- 未来增长。

经验：

```text
小业务：3~6
中等业务：6~12
高吞吐：结合压测决定
```

不要无脑创建大量 partition。

---

## 三、Replication Factor

生产关键 topic 常用：

```text
replication.factor=3
```

配合：

```text
min.insync.replicas=2
acks=all
```

本地单节点只能是 1。

---

## 四、Retention

设计 retention 时考虑：

- consumer 最长停机时间。
- 是否需要历史重放。
- 合规审计。
- 磁盘成本。

订单、支付事件通常保留更久。

---

## 五、容量估算公式

```text
每日数据量 = 平均消息大小 * 每日消息数
总存储 = 每日数据量 * retention 天数 * replication factor
```

示例：

```text
平均消息 2KB
每日 1000 万条
保留 7 天
副本 3

2KB * 1000万 = 约 20GB/天
20GB * 7 * 3 = 420GB
```

还要预留索引、增长和磁盘水位。

---

## 六、Go 后端视角

设计 topic 时要同步设计：

- producer key。
- consumer group。
- retry topic。
- DLQ topic。
- schema version。
- 监控指标。

---

## 七、本节练习

1. 为订单、支付、库存设计 topic。
2. 为每个 topic 选择 partition。
3. 估算 `order.created` 存储容量。
4. 解释 retention 为什么要覆盖恢复窗口。

---

## 八、本节小结

- topic 命名应表达事件事实。
- partition 决定并行度和顺序边界。
- 关键 topic 生产环境通常副本为 3。
- retention 要结合业务恢复窗口。
- 容量估算必须乘副本数。

---

## 九、完整 Topic 设计表

生产方案中建议用表格描述 topic：

| Topic | 业务含义 | Key | Partitions | RF | Retention | Consumer Group |
| --- | --- | --- | --- | --- | --- | --- |
| `order.created` | 订单已创建 | `order_id` | 12 | 3 | 7d | inventory-service, notification-service |
| `order.created.retry.1m` | 订单创建事件 1 分钟重试 | `order_id` | 12 | 3 | 3d | inventory-retry-service |
| `order.created.dlq` | 订单创建死信 | `order_id` | 12 | 3 | 30d | dlq-tool |
| `payment.succeeded` | 支付成功 | `order_id` | 12 | 3 | 30d | order-service |

字段解释：

- RF 是 replication factor。
- retention 要按业务恢复窗口设计。
- DLQ retention 通常比主 topic 更长。

---

## 十、容量估算案例

假设：

```text
order.created 平均消息大小：3KB
日订单量：500 万
峰值系数：3
保留时间：7 天
副本数：3
```

每日原始数据：

```text
3KB * 500万 = 约 15GB/天
```

考虑副本：

```text
15GB * 3 = 45GB/天
```

保留 7 天：

```text
45GB * 7 = 315GB
```

再预留 30%：

```text
315GB * 1.3 = 409.5GB
```

所以这个 topic 至少要按约 410GB 存储预算考虑。

---

## 十一、Partition 数量估算案例

假设库存服务单 consumer 实例稳定处理：

```text
100 msg/s
```

订单峰值：

```text
3000 msg/s
```

理论需要：

```text
3000 / 100 = 30 个并行 consumer
```

那么 partition 至少要支持 30 个并行度。

可设计：

```text
partitions = 36 或 48
```

但还要考虑：

- broker 数量。
- rebalance 成本。
- 是否需要 order_id 顺序。
- 未来增长。

---

## 十二、Retention 设计案例

如果业务要求：

```text
库存服务故障后，最多允许 3 天内恢复并补消费。
```

那么主 topic retention 至少应大于 3 天。

建议：

```text
order.created retention = 7 天
```

DLQ 因为需要人工排查，可能：

```text
order.created.dlq retention = 30 天 或 90 天
```

---

## 十三、Topic 规划常见错误

### 1. 一个 consumer 一个 topic

错误示例：

```text
order.created.inventory
order.created.notification
```

更好：

```text
order.created
```

不同服务使用不同 consumer group。

### 2. partition 设置太少

后续消费能力上不去。

### 3. partition 设置太多

增加 controller 和 rebalance 压力。

### 4. retention 太短

consumer 故障恢复时消息已被清理。

---

## 十四、生产评审问题

上线前回答：

1. 当前 partition 数支持多少 consumer 并行？
2. 峰值写入是多少？
3. 单 consumer 处理能力是多少？
4. retention 是否覆盖故障恢复窗口？
5. DLQ 保留多久？
6. topic 是否需要 compaction？
7. 是否有容量增长预留？

---

## 十四、容量估算例子

假设：

```text
峰值订单：2000 单/秒
每单产生 1 条 order.created
单条消息：2KB
保留时间：7 天
副本数：3
```

每天原始数据量：

```text
2000 * 2KB * 86400 ≈ 345GB
```

考虑 3 副本约 `1TB/天`，保留 7 天约 `7TB`。

---

## 十五、分区估算例子

如果单个 consumer 稳定处理 `100 msg/s`，峰值是 `2000 msg/s`：

```text
至少需要 20 个 consumer 并行
```

topic partition 至少也要接近这个数量，并预留增长空间，例如 `24` 或 `36`。

---

## 十六、估算结论怎么写

不要只写“分区数 24”。要写清楚依据：

```text
峰值 2000 msg/s，单 consumer 约 100 msg/s，因此至少需要 20 个并行消费单元。
考虑增长和 rebalance 余量，order.created 初始设置为 24 partitions。
```

---

## 十七、容量规划表

建议在设计文档里写成表格：

| 项目 | 数值 | 说明 |
| --- | ---: | --- |
| 峰值写入 | 2000 msg/s | 大促峰值 |
| 平均消息大小 | 2 KB | JSON payload |
| retention | 7 天 | 覆盖故障恢复窗口 |
| replication factor | 3 | 生产高可用 |
| 原始数据/天 | 345 GB | 不含副本 |
| 含副本/天 | 1 TB | 3 副本 |
| 7 天存储 | 7 TB | 粗略估算 |

容量估算一定要写明假设，否则数字没有意义。

---

## 十八、常见规划错误

### 1. 只按当前流量估算

topic 分区扩容可以做，但会影响 key 分布和顺序假设。初始设计要预留增长。

### 2. 忽略消息体大小

吞吐不是只有 msg/s。2KB 消息和 200KB 消息对网络、磁盘、压缩的影响完全不同。

### 3. retention 拍脑袋

retention 要覆盖 consumer 最长可接受停机时间和人工修复窗口。

---

## 十九、设计结论模板

```text
order.created:
  partitions: 24
  replication.factor: 3
  retention: 7d
  key: order_id
  reason:
    峰值 2000 msg/s，单 consumer 100 msg/s，预留增长后选择 24 partition。
    retention 7d 覆盖库存服务最长 2 天故障和人工修复窗口。
```

结论要写理由，方便后续评审。

---

## 二十、评审追问

评审时别人可能会问：

```text
为什么不是 12 个 partition？
为什么 retention 不是 3 天？
如果消息大小翻倍怎么办？
如果 consumer 单实例只能处理 50 msg/s 怎么办？
```

容量设计要能经得起这些追问。

---

## 二十一、最终验收

完成本节后，应该能把一个 topic 的容量估算讲成：

```text
流量假设 -> 消息大小 -> 保留时间 -> 副本数 -> 存储成本 -> partition 数
```

---

## 二十一、容量估算示例

可以按下面方式粗估：

```text
订单创建峰值：1000 条/秒
单条消息大小：2KB
每天数据量：1000 * 2KB * 86400 ≈ 172.8GB
保留时间：3 天
副本数：3
粗略存储需求：172.8GB * 3 * 3 ≈ 1.55TB
```

这个估算没有考虑压缩和索引文件，但足够帮助你判断 topic 规划不是随手写一个名字。

生产设计里，流量假设必须写出来，否则 partition 和保留时间都没有依据。
