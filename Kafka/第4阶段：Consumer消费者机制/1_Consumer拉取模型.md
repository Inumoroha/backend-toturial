# 1. Consumer 拉取模型

本节目标：理解 Kafka consumer 如何从 broker 拉取消息，知道 poll/fetch 模型与 Go 后端后台 worker 的关系。

Kafka consumer 不是 broker 主动把消息推给你的 Go 服务，而是 consumer 主动从 broker 拉取消息。理解这一点后，才能理解为什么 consumer 处理太慢会导致 lag 增长，为什么 poll 间隔过长会触发 rebalance。

---

## 一、Consumer 在 Go 后端中的位置

一个典型 consumer 服务：

```text
inventory-service
  启动 Kafka consumer
  订阅 order.created
  拉取消息
  扣减库存
  提交 offset
```

它通常不是 HTTP 请求里的短生命周期逻辑，而是一个长期运行的后台 worker。

常见入口：

```text
cmd/inventory-service/main.go
```

业务处理：

```text
internal/inventory/handler.go
```

Kafka 封装：

```text
internal/kafka/consumer.go
```

---

## 二、拉取模型是什么

Kafka consumer 的基本流程：

```text
连接 broker
加入 consumer group
获取 partition 分配
循环 poll/fetch 消息
处理消息
提交 offset
继续 poll/fetch
```

注意：

```text
consumer 主动拉取消息
```

不是 broker 主动推消息。

---

## 三、为什么 Kafka 使用拉取

拉取模型的好处：

- consumer 可以按自己的处理能力读取。
- consumer 可以批量拉取。
- consumer 可以控制 offset。
- 慢 consumer 不会被 broker 强行推爆。

缺点：

- consumer 需要合理控制 poll 频率。
- 处理太慢会导致 lag。
- 长时间不 poll 可能被认为不健康。

---

## 四、Poll/Fetch 直觉

不同客户端 API 名称不同，但核心都是：

```text
向 broker 请求一批消息
```

拉取到的可能是：

- 0 条消息。
- 1 条消息。
- 多条消息。
- 错误。

Consumer 主循环通常类似：

```text
for {
    fetch messages
    for each message:
        handle
        commit if success
}
```

---

## 五、单条处理与批量处理

### 单条处理

```text
拉一条
处理一条
提交一条
```

优点：

- 逻辑简单。
- 失败边界清晰。

缺点：

- 吞吐较低。

适合：

- 订单。
- 支付。
- 库存。

### 批量处理

```text
拉一批
批量处理
提交批次 offset
```

优点：

- 吞吐高。
- 适合批量写数据库。

缺点：

- 批次中某条失败时处理复杂。

适合：

- 日志写入。
- 行为数据。
- 统计聚合。

---

## 六、Consumer 拉取不是业务成功

拉到消息只表示：

```text
consumer 从 Kafka 取到了消息
```

不表示：

```text
业务已经处理成功
```

更不表示：

```text
offset 已经安全提交
```

Go 后端中必须区分：

- fetch 成功。
- decode 成功。
- handler 成功。
- offset commit 成功。

这些是不同步骤。

---

## 七、命令行体验拉取

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage4.consumer.fetch \
  --partitions 3 \
  --replication-factor 1
```

先启动 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage4.consumer.fetch \
  --group fetch-demo-group
```

此时如果没有消息，consumer 会等待。

另一个终端发送：

```bash
echo "hello-consumer" | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage4.consumer.fetch
```

consumer 会打印消息。

这就是 consumer 主动等待并拉取消息的直觉。

---

## 八、Go Consumer 主循环伪代码

```go
for ctx.Err() == nil {
    records, err := consumer.Fetch(ctx)
    if err != nil {
        log.Error("fetch failed", "err", err)
        continue
    }

    for _, record := range records {
        err := handler.Handle(ctx, record)
        if err != nil {
            handleError(ctx, record, err)
            continue
        }

        err = consumer.Commit(ctx, record)
        if err != nil {
            log.Error("commit failed", "topic", record.Topic, "offset", record.Offset)
        }
    }
}
```

这段伪代码强调：

```text
先处理业务，再提交 offset
```

---

## 九、Consumer 日志字段

每条消息处理日志应该包含：

```text
topic
partition
offset
key
event_id
group
handler
duration_ms
error
```

示例：

```text
level=info msg="message handled" topic=order.created partition=1 offset=203 key=order_1001 event_id=evt_1 group=inventory-service duration_ms=35
```

没有这些字段，线上排查会很痛苦。

---

## 十、常见误区

### 1. Consumer 是被 Kafka 推送消息

不是。

consumer 主动拉取。

### 2. 拉到消息就可以提交 offset

不是。

业务成功后再提交。

### 3. 批量处理一定更好

不一定。

关键业务更看重失败边界和幂等。

### 4. Consumer 没消息就说明出问题

不一定。

可能 topic 里没有新消息，consumer 正在等待。

---

## 十一、本节练习

1. 启动一个 consumer，让它等待消息。
2. 在另一个终端发送消息，观察 consumer 输出。
3. 写出 consumer 主循环伪代码。
4. 说明 fetch 成功和业务成功的区别。
5. 设计订单 consumer 的日志字段。

---

## 十二、本节小结

- Kafka consumer 主动从 broker 拉取消息。
- 拉取模型让 consumer 可以按自身能力处理。
- 拉到消息不等于业务成功。
- 关键业务通常先单条处理，后续再考虑批量优化。
- Go consumer 应该有清晰主循环。
- 日志必须记录 topic、partition、offset、key 和 event_id。

---

## 十三、Go Consumer 主循环伪代码

```go
for {
    msg, err := consumer.Fetch(ctx)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            return nil
        }
        logger.Error("fetch message failed", "error", err)
        continue
    }

    start := time.Now()
    err = handler.Handle(ctx, msg)
    duration := time.Since(start)

    logger.Info("message handled",
        "topic", msg.Topic,
        "partition", msg.Partition,
        "offset", msg.Offset,
        "duration_ms", duration.Milliseconds(),
    )

    if err != nil {
        return err
    }
}
```

这里还没有提交 offset。提交策略会在后续章节展开。

---

## 十四、拉取成功不等于处理成功

一定要区分：

```text
Fetch 成功：消息从 broker 拉到了 consumer。
Handle 成功：业务逻辑处理完成。
Commit 成功：consumer group 进度更新。
```

可靠消费要把这三个动作分开看。

---

## 十五、最终练习

写出下面三个动作的日志：

```text
fetch_success
handle_success
commit_success
```

每条日志都带上 topic、partition、offset。

这能帮助你在后续章节理解消息到底卡在哪一步。

---

## 十六、排查提示

如果 consumer 没有输出，不一定是坏了。可能只是 broker 里暂时没有新消息，consumer 正在长轮询等待。

---

## 十七、观察 poll 的三个指标

学习拉取模型时，建议同时观察：

| 指标 | 含义 | 异常时可能说明 |
| --- | --- | --- |
| 拉取批次大小 | 每次 poll 返回多少消息 | batch 太小或流量太低 |
| handler 耗时 | 业务处理一条消息多久 | 下游数据库或接口慢 |
| poll 间隔 | 两次拉取之间相隔多久 | 处理太慢或线程被阻塞 |

这三个指标能帮助你判断 consumer 是“没有消息可拉”，还是“拉到了但处理不过来”。

后续看 lag 时，也要结合 handler 耗时一起判断。
