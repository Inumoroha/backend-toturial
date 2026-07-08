# 6. 综合实践：观察 Partition 与 Offset 变化

本节目标：通过一个完整实验观察 topic、partition、key、offset、consumer group、lag 的变化，把第 2 阶段的核心概念串起来。

这一节不引入太多新概念，重点是通过终端输出形成直觉。你应该像做 PostgreSQL 的 SQL 练习一样，把每个命令跑一遍，记录输出。

---

## 一、实验目标

完成以下观察：

```text
创建 3 partition topic
发送带 key 的订单消息
观察消息分布到哪个 partition
使用 consumer group 消费
查看 offset 和 lag
停止 consumer 后继续发送
观察 lag 增长
重启 consumer 后观察 lag 下降
```

---

## 二、创建实验 Topic

进入 Kafka 容器：

```powershell
docker exec -it kafka bash
```

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage2.practice.orders \
  --partitions 3 \
  --replication-factor 1
```

查看详情：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic stage2.practice.orders
```

记录：

- partition 数量。
- 每个 partition 的 leader。
- replicas。
- ISR。

---

## 三、发送带 Key 的消息

启动 producer：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.practice.orders \
  --property parse.key=true \
  --property key.separator=:
```

输入：

```text
order-1:created
order-1:paid
order-2:created
order-2:paid
order-3:created
order-3:paid
order-1:shipped
```

退出 producer。

---

## 四、打印 Partition 与 Offset

启动 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.practice.orders \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true \
  --timeout-ms 10000
```

记录每条消息的：

- key。
- partition。
- offset。

观察：

- `order-1` 的多条消息是否在同一个 partition。
- offset 是否在每个 partition 内递增。

---

## 五、使用 Consumer Group 消费

启动带 group 的 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.practice.orders \
  --group stage2-practice-group \
  --from-beginning
```

消费完后停止。

查看 group：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group stage2-practice-group
```

记录：

- `CURRENT-OFFSET`
- `LOG-END-OFFSET`
- `LAG`

---

## 六、制造 Lag

确保 group consumer 已经停止。

继续发送消息：

```bash
for i in $(seq 4 10); do
  echo "order-$i:created"
done | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.practice.orders \
  --property parse.key=true \
  --property key.separator=:
```

查看 group：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group stage2-practice-group
```

预期：

```text
LAG 大于 0
```

---

## 七、恢复消费

重新启动 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.practice.orders \
  --group stage2-practice-group
```

消费完后再次查看 lag。

预期：

```text
LAG 回到 0
```

---

## 八、实验现象解释

### 1. 为什么相同 key 进入同一 partition？

producer 根据 key 选择 partition。

这让同一业务对象的消息可以保持 partition 内顺序。

### 2. 为什么不同 partition 的 offset 都可能从 0 开始？

offset 是 partition 内的位置，不是 topic 全局位置。

### 3. 为什么 consumer 停止后 lag 增长？

producer 继续写入，log end offset 增长。

consumer group 没有提交新 offset，所以 lag 增长。

### 4. 为什么重启 consumer 后能继续消费？

Kafka 记录了 group 在每个 partition 上的 offset。

---

## 九、Go 后端工程视角

这个实验对应真实服务：

```text
order-service -> order.created -> inventory-service
```

如果 `inventory-service` 停止：

- `order-service` 仍然可以写 Kafka。
- Kafka 保存消息。
- `inventory-service` 的 lag 增长。
- 恢复后从上次 offset 继续消费。

但注意：

```text
如果停机时间超过 retention，历史消息可能被清理
```

所以生产环境要监控 lag，并设计合理 retention。

---

## 十、实验报告模板

```markdown
# 第 2 阶段综合实践报告

## Topic 配置

## Producer 输入

## Consumer 输出

## Partition 分布

## Offset 观察

## Lag 变化

## 我的结论

## 仍然不懂的问题
```

---

## 十一、本节练习

1. 把 topic partition 数改成 6，重新做实验。
2. 比较相同 key 的 partition 分布是否变化。
3. 使用另一个 group `stage2-another-group` 从头消费。
4. 对比两个 group 的 offset。
5. 写一份实验报告。

---

## 十二、本节小结

- partition 是 Kafka 并行和顺序的边界。
- key 决定消息如何分布到 partition。
- offset 在 partition 内递增。
- consumer group 维护自己的 offset。
- lag 可以通过命令观察。
- producer 和 consumer 解耦，consumer 停止不影响 producer 写入。
- retention 决定 Kafka 能为落后 consumer 保留多久的数据。

---

## 十四、实验记录模板

建议每次实验都记录：

```text
topic:
partitions:
producer key:
message count:
consumer group:
observed partitions:
latest offsets:
group lag:
```

例如：

```text
topic=order.created
partitions=3
key=order_id
message_count=10
group=inventory-service
lag=0
```

记录这些信息，才能把“我看到了现象”变成“我理解了机制”。

---

## 十五、进一步观察

可以再做两组对比：

1. 使用相同 key 连续发送 10 条消息。
2. 使用不同 key 连续发送 10 条消息。

观察它们落到的 partition 是否不同。

这会帮助你理解为什么订单事件通常用 `order_id` 作为 key。

---

## 十六、结果解释模板

完成实验后，可以这样写结论：

```text
同一个 key 的消息会稳定进入同一个 partition，因此 offset 在该 partition 内连续递增。
不同 key 的消息会根据分区策略分散到不同 partition。
consumer group 的 lag 是按 partition 计算的，不是 topic 全局一个数字。
```

---

## 十七、常见异常

如果观察不到 partition 信息，检查 consumer 命令是否开启了打印配置。

如果换 group 后仍然读不到旧消息，检查：

- 是否使用 `--from-beginning`。
- topic 中消息是否被 retention 清理。
- group 名是否真的不同。

---

## 十八、最终检查

请用一段话解释：

```text
为什么同一个 topic 下，不同 consumer group 可以各自维护自己的消费进度？
```

这句话能说清，说明 partition、offset、group 的关系已经串起来了。

---

## 十九、实验交付物

最终提交一份实验记录：

```text
topic describe 输出。
producer 输入样例。
consumer 输出样例。
group describe 输出。
lag 变化说明。
```

这些输出能证明你不是只理解概念，而是真的观察过 Kafka 状态变化。

---

## 二十、最终验收

实验结束后，用自己的话解释：

```text
同一个 key 为什么会进入同一个 partition？
为什么 offset 只在 partition 内递增？
为什么 group lag 要按 partition 看？
```

这三个问题能回答清楚，说明本阶段实践达标。

---

## 二十一、实践结论要写具体

本节实验结束后，结论不要写成“观察到了 offset 变化”。

更好的写法是：

```text
topic 有 3 个 partition。
同一个 key 的消息进入同一个 partition。
不同 key 的消息可能分布到不同 partition。
每个 partition 的 offset 独立递增。
同一个 consumer group 对每个 partition 分别记录 offset。
新 group 从配置的 auto.offset.reset 位置开始消费。
```

这种写法能直接服务后续学习：Producer 讨论 key，Consumer 讨论 group，可靠性阶段讨论重复消费。
