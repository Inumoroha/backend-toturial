# 05 Producer 与 Consumer 实验课

本节目标：把 producer 参数、message key、手动提交、失败处理和 rebalance 用实验串起来。

## 实验前准备

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic lab.producer.consumer \
  --partitions 3 \
  --replication-factor 1
```

## 实验 1：观察 Key 如何影响 Partition

启动 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.producer.consumer \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true \
  --property key.separator=" | "
```

启动 producer：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.producer.consumer \
  --property parse.key=true \
  --property key.separator=:
```

输入：

```text
order-1:created-1
order-1:paid-1
order-2:created-2
order-2:paid-2
order-1:shipped-1
```

观察：

- 相同 key 通常进入同一个 partition。
- 同一个 partition 内 offset 递增。

结论：

```text
如果同一订单的事件必须有序，message key 应该使用 order_id
```

## 实验 2：Consumer Group Rebalance

创建新 group：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.producer.consumer \
  --group lab-rebalance-service \
  --property print.partition=true \
  --property print.offset=true
```

再打开第二个、第三个相同命令的 consumer。

观察：

- 新 consumer 加入时，已有 consumer 可能短暂停顿。
- partition 会重新分配。

然后关闭其中一个 consumer。

观察：

- 剩余 consumer 会接管它的 partition。

Go 后端含义：

- consumer 处理逻辑必须允许重复消费。
- consumer 退出要优雅，减少不必要 rebalance。

## 实验 3：Lag 增长与下降

停止所有 `lab-rebalance-service` consumer。

批量写入消息：

```bash
for i in $(seq 1 100); do
  echo "order-$i:created-$i"
done | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic lab.producer.consumer \
  --property parse.key=true \
  --property key.separator=:
```

查看 lag：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group lab-rebalance-service
```

重新启动 consumer，再次查看 lag。

预期：

- consumer 停止时 lag 增长。
- consumer 恢复后 lag 下降。

## 实验 4：模拟坏消息

向 topic 发送一条格式错误消息：

```text
bad-message-without-json
```

然后思考 consumer 应该如何处理：

| 处理方式 | 后果 |
| --- | --- |
| 不提交 offset，持续重试 | 当前 partition 被卡住 |
| 直接提交 offset | 坏消息被跳过，但没有记录 |
| 写入 DLQ 后提交 offset | 主链路继续，坏消息可追踪 |

生产建议：

```text
不可解析消息 -> 写 DLQ 成功 -> 提交原 offset
```

## 实验 5：可靠发送配置理解

命令行 producer 不适合完整演示所有生产配置，但你需要理解配置取舍：

| 目标 | 配置倾向 |
| --- | --- |
| 不丢关键事件 | `acks=all`、开启幂等、合理重试 |
| 高吞吐日志 | 批量、压缩、适当 linger |
| 低延迟 | 小 linger、控制 batch |

Go 项目中，producer 配置必须按业务类型区分，不能全局一套配置打天下。

## 本节验收

你需要能写出下面问题的答案：

1. 为什么 `order_id` 适合做订单事件 key？
2. consumer rebalance 时为什么可能重复消费？
3. lag 增长一定代表 Kafka 出问题吗？
4. 坏消息为什么不能无限重试？
5. 写入 retry topic 或 DLQ 成功后，为什么可以提交原 offset？

