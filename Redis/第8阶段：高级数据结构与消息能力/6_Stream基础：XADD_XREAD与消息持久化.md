# 6. Stream 基础：XADD、XREAD 与消息持久化

Redis Stream 是 Redis 5.0 引入的消息流结构。

它比 Pub/Sub 更可靠，因为消息会保存在 Stream 中，消费者可以按 ID 读取历史消息。

学完这一节后，你应该能够：

- 理解 Stream 的基本模型。
- 使用 `XADD` 写入消息。
- 使用 `XREAD` 读取消息。
- 理解消息 ID。
- 在 Go 中写入和读取 Stream。
- 知道 Stream 的基本清理方式。

---

## 一、Stream 是什么

Stream 可以理解为一个追加写入的消息日志。

每条消息都有 ID：

```text
1790000000000-0
1790000000001-0
1790000000001-1
```

消息内容是 field-value：

```text
type=created
order_id=1001
user_id=2001
```

写入后，消息会保留在 Redis 中，直到被修剪或删除。

---

## 二、写入消息 XADD

```redis
XADD order:events * type created order_id 1001 user_id 2001
```

含义：

- `order:events` 是 Stream key。
- `*` 表示让 Redis 自动生成 ID。
- 后面是字段和值。

返回：

```text
1790000000000-0
```

这是消息 ID。

---

## 三、消息 ID

Stream ID 格式：

```text
milliseconds-sequence
```

例如：

```text
1790000000000-0
```

前半部分通常是毫秒时间戳。

后半部分是同一毫秒内的序号。

通常让 Redis 用 `*` 自动生成 ID。

不要自己随便生成，除非你明确知道 ID 递增规则。

---

## 四、读取消息 XREAD

从头读取：

```redis
XREAD COUNT 10 STREAMS order:events 0
```

读取某个 ID 之后的消息：

```redis
XREAD COUNT 10 STREAMS order:events 1790000000000-0
```

阻塞等待新消息：

```redis
XREAD BLOCK 5000 COUNT 10 STREAMS order:events $
```

`$` 表示从当前最新位置开始，只读之后的新消息。

---

## 五、XREAD 和 Pub/Sub 的区别

Pub/Sub：

```text
只给在线订阅者推消息。
```

Stream：

```text
消息写入 Stream，消费者可以之后读取。
```

如果消费者短暂离线：

- Pub/Sub 消息会丢。
- Stream 消息还在，可以补读。

这就是 Stream 更适合任务队列的原因。

---

## 六、Go 写入 Stream

```go
func AddOrderEvent(ctx context.Context, rdb *redis.Client, orderID, userID int64, eventType string) (string, error) {
    return rdb.XAdd(ctx, &redis.XAddArgs{
        Stream: "order:events",
        Values: map[string]interface{}{
            "type":     eventType,
            "order_id": orderID,
            "user_id":  userID,
        },
    }).Result()
}
```

调用：

```go
id, err := AddOrderEvent(ctx, rdb, 1001, 2001, "created")
```

返回的 `id` 可以用于日志追踪。

---

## 七、Go XREAD 读取

```go
func ReadOrderEvents(ctx context.Context, rdb *redis.Client, lastID string) ([]redis.XStream, error) {
    return rdb.XRead(ctx, &redis.XReadArgs{
        Streams: []string{"order:events", lastID},
        Count:   10,
        Block:   5 * time.Second,
    }).Result()
}
```

处理：

```go
lastID := "0"

for {
    streams, err := ReadOrderEvents(ctx, rdb, lastID)
    if err != nil {
        if errors.Is(err, redis.Nil) {
            continue
        }
        log.Printf("xread failed err=%v", err)
        continue
    }

    for _, stream := range streams {
        for _, msg := range stream.Messages {
            handle(msg)
            lastID = msg.ID
        }
    }
}
```

这只是单消费者示例。

生产中更常用消费者组。

---

## 八、消息内容设计

Stream 消息建议保存轻量字段：

```text
type = created
order_id = 1001
user_id = 2001
trace_id = abc
```

不要把巨大 JSON 塞进 Stream。

如果消息内容很复杂：

- Stream 里保存 task_id。
- 详情放数据库。
- 消费者根据 task_id 查询。

这样更容易重试、审计和修复。

---

## 九、Stream 长度控制

Stream 会保存消息。

如果不清理，会越来越大。

写入时可以限制长度：

```redis
XADD order:events MAXLEN ~ 100000 * type created order_id 1001
```

`~` 表示近似修剪，性能更好。

Go：

```go
rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "order:events",
    MaxLenApprox: 100000,
    Values: map[string]interface{}{
        "type": "created",
    },
})
```

---

## 十、Stream 适合什么场景

适合：

- 轻量异步任务。
- 订单事件流。
- 缓存重建任务。
- 操作日志缓冲。
- 低到中等规模消息处理。

不适合：

- 超高吞吐日志平台。
- 跨数据中心复杂消息系统。
- 大规模分区消费。
- 复杂路由和死信治理。

这些更适合 Kafka、RabbitMQ 等。

---

## 十一、常见错误

### 1. Stream 不限制长度

消息会无限堆积。

### 2. 消息体太大

会增加 Redis 内存和网络压力。

### 3. 用 `$` 启动消费者后以为能读历史消息

`$` 只读之后的新消息。

### 4. 单消费者自己维护 lastID 不可靠

服务重启后需要保存消费位置。

### 5. 以为 Stream 等于完整 MQ

Stream 很实用，但复杂消息系统仍需专业 MQ。

---

## 十二、本节练习

请完成下面练习：

1. 写出 `XADD` 添加订单创建消息。
2. 写出 `XREAD` 从头读取消息。
3. 写出 `XREAD BLOCK` 等待新消息。
4. 用 Go 实现 `AddOrderEvent`。
5. 用 Go 实现简单 `XREAD` 循环。
6. 解释 Stream ID 的格式。
7. 说明为什么要限制 Stream 长度。
8. 判断订单事件是否适合用 Stream 起步。

---

## 十三、本节小结

这一节你学习了 Stream 基础。

你需要记住：

- Stream 是可持久化的消息流。
- `XADD` 写入消息，`XREAD` 读取消息。
- 消息 ID 通常由 Redis 自动生成。
- Stream 比 Pub/Sub 更可靠，因为消息会保留。
- Stream 要控制长度，避免无限增长。

下一节我们继续学习消费者组、ACK 和 Pending List。

