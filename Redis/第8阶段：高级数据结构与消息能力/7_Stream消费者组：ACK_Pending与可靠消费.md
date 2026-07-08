# 7. Stream 消费者组：ACK、Pending 与可靠消费

上一节学习了 `XADD` 和 `XREAD`。

这一节进入 Stream 更重要的能力：消费者组。

消费者组可以让多个消费者协同处理消息，并通过 ACK 和 Pending List 跟踪未确认消息。

学完这一节后，你应该能够：

- 创建消费者组。
- 使用 `XREADGROUP` 消费消息。
- 使用 `XACK` 确认消息。
- 理解 Pending List。
- 处理消费者崩溃后的未确认消息。
- 在 Go 中实现一个基础消费者。

---

## 一、为什么需要消费者组

单消费者读取 Stream：

```text
只有一个 worker 处理所有消息。
```

如果消息量变大，需要多个 worker：

```text
worker-1
worker-2
worker-3
```

消费者组让同一个组里的多个消费者分摊消息。

每条新消息通常只会被组内一个消费者领取。

---

## 二、创建消费者组

```redis
XGROUP CREATE order:events order-workers 0 MKSTREAM
```

含义：

- `order:events`：Stream key。
- `order-workers`：消费者组名。
- `0`：从头开始消费。
- `MKSTREAM`：如果 Stream 不存在就创建。

如果只想消费创建组之后的新消息：

```redis
XGROUP CREATE order:events order-workers $ MKSTREAM
```

---

## 三、消费新消息

```redis
XREADGROUP GROUP order-workers consumer-1 COUNT 10 BLOCK 5000 STREAMS order:events >
```

关键参数：

```text
GROUP order-workers consumer-1
```

表示消费者组和消费者名。

`>` 表示只读取还没有投递给其他消费者的新消息。

---

## 四、确认消息 XACK

消费者处理成功后，要确认：

```redis
XACK order:events order-workers 1790000000000-0
```

确认后，这条消息会从 Pending List 中移除。

如果不 ACK：

```text
Redis 会认为这条消息已经投递，但还没处理完成。
```

它会留在 Pending List。

---

## 五、Pending List 是什么

Pending List 记录：

```text
已经投递给消费者，但还没有 ACK 的消息。
```

查询：

```redis
XPENDING order:events order-workers
```

可以看到：

- Pending 总数。
- 最小消息 ID。
- 最大消息 ID。
- 每个消费者 Pending 数量。

查看详情：

```redis
XPENDING order:events order-workers - + 10
```

---

## 六、消费者崩溃会发生什么

时间线：

```text
consumer-1 读取消息 A
consumer-1 处理到一半崩溃
消息 A 没有 XACK
消息 A 留在 Pending List
```

这时新消费者不会通过 `>` 再读到消息 A。

需要主动认领 Pending 消息。

Redis 提供：

- `XCLAIM`
- `XAUTOCLAIM`

新版本更推荐 `XAUTOCLAIM`。

---

## 七、自动认领 Pending 消息

```redis
XAUTOCLAIM order:events order-workers consumer-2 60000 0-0 COUNT 10
```

含义：

```text
把空闲超过 60 秒的 Pending 消息认领给 consumer-2。
```

适合处理崩溃消费者留下的消息。

注意：被重新认领的消息可能已经被原消费者处理了一部分。

所以业务处理必须幂等。

---

## 八、Go 创建消费者组

```go
func CreateGroup(ctx context.Context, rdb *redis.Client, stream, group string) error {
    err := rdb.XGroupCreateMkStream(ctx, stream, group, "0").Err()
    if err != nil && !strings.Contains(err.Error(), "BUSYGROUP") {
        return err
    }
    return nil
}
```

如果消费者组已存在，Redis 会返回 `BUSYGROUP`。

这通常可以忽略。

---

## 九、Go 消费消息

```go
func ReadGroup(ctx context.Context, rdb *redis.Client, stream, group, consumer string) ([]redis.XMessage, error) {
    streams, err := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
        Group:    group,
        Consumer: consumer,
        Streams:  []string{stream, ">"},
        Count:    10,
        Block:    5 * time.Second,
    }).Result()
    if err != nil {
        return nil, err
    }
    if len(streams) == 0 {
        return nil, nil
    }
    return streams[0].Messages, nil
}
```

处理后 ACK：

```go
func Ack(ctx context.Context, rdb *redis.Client, stream, group, id string) error {
    return rdb.XAck(ctx, stream, group, id).Err()
}
```

---

## 十、基础 worker

```go
func RunStreamWorker(ctx context.Context, rdb *redis.Client, stream, group, consumer string, handler func(context.Context, redis.XMessage) error) error {
    if err := CreateGroup(ctx, rdb, stream, group); err != nil {
        return err
    }

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        msgs, err := ReadGroup(ctx, rdb, stream, group, consumer)
        if err != nil {
            if errors.Is(err, redis.Nil) {
                continue
            }
            log.Printf("xreadgroup failed stream=%s group=%s consumer=%s err=%v", stream, group, consumer, err)
            continue
        }

        for _, msg := range msgs {
            if err := handler(ctx, msg); err != nil {
                log.Printf("handle stream msg failed id=%s err=%v", msg.ID, err)
                continue
            }

            if err := Ack(ctx, rdb, stream, group, msg.ID); err != nil {
                log.Printf("xack failed id=%s err=%v", msg.ID, err)
            }
        }
    }
}
```

处理失败时不 ACK，消息会留在 Pending。

后续可以由重试逻辑认领。

---

## 十一、业务处理必须幂等

Stream 至少可能投递一次。

也就是说：

```text
同一条消息可能被处理多次。
```

原因：

- handler 成功但 ACK 失败。
- 消费者处理成功后崩溃。
- Pending 消息被其他消费者认领。
- 网络重试。

所以消费逻辑必须幂等。

例如订单状态更新：

```sql
UPDATE orders
SET status = 'paid'
WHERE id = ?
  AND status = 'pending';
```

重复执行不会破坏状态。

---

## 十二、Pending 监控

生产中要监控：

- Pending 总数。
- 每个消费者 Pending 数量。
- 最老 Pending 消息空闲时间。
- 消息处理失败次数。
- ACK 失败次数。
- Stream 长度。

Pending 积压通常说明：

- 消费者崩溃。
- handler 太慢。
- 下游服务异常。
- 消息重试卡住。

---

## 十三、什么时候删除消息

`XACK` 只是从 Pending List 移除。

它不会自动从 Stream 删除消息。

Stream 消息仍然存在，直到：

- `XDEL` 删除。
- `XTRIM` 修剪。
- `XADD MAXLEN` 写入时修剪。

通常更推荐按长度或时间保留一段消息，方便排查。

---

## 十四、常见错误

### 1. 处理成功后忘记 XACK

Pending 会持续增长。

### 2. 以为 XACK 会删除 Stream 消息

ACK 只是确认消费，不等于删除消息。

### 3. 业务消费不幂等

消息重复投递会造成重复发券、重复扣款等问题。

### 4. 不处理 Pending

消费者崩溃后，消息会一直卡在 Pending。

### 5. 不监控 Stream 长度和 Pending

消息系统出问题时难以及时发现。

---

## 十五、本节练习

请完成下面练习：

1. 写出创建消费者组的命令。
2. 写出使用 `XREADGROUP` 读取新消息的命令。
3. 写出 `XACK` 确认消息的命令。
4. 查询 Pending 概览。
5. 用 Go 实现 `CreateGroup`。
6. 用 Go 实现基础 worker。
7. 解释为什么消费逻辑必须幂等。
8. 说明 `XACK` 和删除消息的区别。

---

## 十六、本节小结

这一节你学习了 Stream 消费者组。

你需要记住：

- 消费者组让多个消费者分摊消息。
- `XREADGROUP ... >` 读取新消息。
- 消费成功后必须 `XACK`。
- 未 ACK 的消息会进入 Pending List。
- 崩溃消费者留下的消息需要重新认领。
- Stream 消费是至少一次投递，业务必须幂等。

下一节我们会比较 Stream、Kafka 和 RabbitMQ，理解什么时候该换专业 MQ。

