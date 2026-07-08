# 5. Pub/Sub：轻量通知与不可靠投递边界

Redis Pub/Sub 是发布订阅模型。

它使用起来非常简单，但可靠性很弱。理解这个边界，比会写命令更重要。

学完这一节后，你应该能够：

- 使用 `PUBLISH`、`SUBSCRIBE`。
- 在 Go 中发布和订阅消息。
- 理解 Pub/Sub 的消息丢失问题。
- 判断哪些场景适合 Pub/Sub。
- 知道什么时候应该改用 Stream 或专业 MQ。

---

## 一、Pub/Sub 是什么

发布者把消息发到频道：

```redis
PUBLISH notice "hello"
```

订阅者订阅频道：

```redis
SUBSCRIBE notice
```

只要订阅者在线，就能收到消息。

它像一个广播喇叭：

```text
正在听的人能听到；
不在线的人听不到。
```

---

## 二、基本命令

打开一个客户端订阅：

```redis
SUBSCRIBE cache:invalid
```

另一个客户端发布：

```redis
PUBLISH cache:invalid article:1001
```

订阅端会收到：

```text
channel = cache:invalid
message = article:1001
```

---

## 三、模式订阅

可以按模式订阅：

```redis
PSUBSCRIBE cache:*
```

这会收到：

```text
cache:invalid
cache:reload
cache:notice
```

模式订阅适合开发和内部通知。

生产中不要滥用过宽模式，避免消息流难以控制。

---

## 四、Go 发布消息

```go
func PublishCacheInvalid(ctx context.Context, rdb *redis.Client, key string) error {
    return rdb.Publish(ctx, "cache:invalid", key).Err()
}
```

调用：

```go
err := PublishCacheInvalid(ctx, rdb, "article:detail:1001")
```

可以用于通知其他实例删除本地缓存。

---

## 五、Go 订阅消息

```go
func SubscribeCacheInvalid(ctx context.Context, rdb *redis.Client, handler func(string)) error {
    pubsub := rdb.Subscribe(ctx, "cache:invalid")
    defer pubsub.Close()

    ch := pubsub.Channel()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case msg := <-ch:
            if msg == nil {
                continue
            }
            handler(msg.Payload)
        }
    }
}
```

handler 示例：

```go
func(key string) {
    localCache.Delete(key)
}
```

这样一个实例更新数据后，可以通知其他实例清理本地缓存。

---

## 六、Pub/Sub 最大的问题

Pub/Sub 不保存消息。

如果订阅者不在线：

```text
消息直接丢失。
```

如果订阅者断线：

```text
断线期间的消息收不到。
```

如果订阅者处理失败：

```text
没有 ACK，也没有重试。
```

这就是它和消息队列最大的区别。

---

## 七、适合 Pub/Sub 的场景

适合：

- 本地开发通知。
- 多实例本地缓存失效通知。
- 非关键广播。
- 在线用户临时通知。
- 简单配置变更提醒。

这些场景通常可以接受：

```text
偶尔丢一条消息，系统仍然能靠 TTL 或后台刷新恢复。
```

例如本地缓存失效：

- Pub/Sub 消息丢了。
- 本地缓存最多旧一段时间。
- 本地缓存 TTL 到期后自然恢复。

这种风险可控。

---

## 八、不适合 Pub/Sub 的场景

不适合：

- 订单创建消息。
- 支付成功消息。
- 发货通知。
- 扣库存消息。
- 积分发放。
- 必须重试的任务。
- 需要消息回放的业务。

这些场景需要：

- 持久化。
- ACK。
- 重试。
- 死信。
- 消费进度。

应该考虑 Redis Stream、Kafka、RabbitMQ 等。

---

## 九、缓存失效通知示例

场景：

```text
服务 A 更新文章 1001。
服务 A 删除 Redis 缓存。
服务 A 发布 Pub/Sub 通知。
服务 B、C 收到通知后删除本地缓存。
```

流程：

```go
func (s *ArticleService) UpdateArticle(ctx context.Context, article *Article) error {
    if err := s.repo.Update(ctx, article); err != nil {
        return err
    }

    cacheKey := fmt.Sprintf("article:detail:%d", article.ID)
    _ = s.rdb.Del(ctx, cacheKey).Err()
    _ = s.rdb.Publish(ctx, "cache:invalid", cacheKey).Err()

    return nil
}
```

注意：发布失败要记录日志。

本地缓存还应该有 TTL 兜底。

---

## 十、订阅连接管理

订阅通常是长连接。

要注意：

- 服务启动时创建订阅 goroutine。
- 服务退出时关闭订阅。
- 订阅断开要重连。
- handler 不要长时间阻塞。
- 处理逻辑要 recover，避免 goroutine 退出。

教学示例很短，生产代码需要更完整的生命周期管理。

---

## 十一、Pub/Sub 和 Stream 的区别

| 能力 | Pub/Sub | Stream |
| --- | --- | --- |
| 是否持久化 | 否 | 是 |
| 离线后能否补消息 | 否 | 可以 |
| ACK | 无 | 有 |
| 消费者组 | 无 | 有 |
| 适合场景 | 轻量广播 | 轻量消息队列 |

如果你需要可靠消费，直接考虑 Stream。

不要试图给 Pub/Sub 手写一套 ACK。

---

## 十二、常见错误

### 1. 用 Pub/Sub 做订单消息

订阅者不在线时消息会丢。

### 2. 订阅 handler 里做慢操作

可能阻塞消息处理。

### 3. 没有本地缓存 TTL 兜底

缓存失效消息丢失后，本地缓存可能长期不刷新。

### 4. 不处理订阅断线

服务以为自己还在订阅，实际已经收不到消息。

### 5. 以为 Publish 成功等于消费者处理成功

Pub/Sub 没有消费者确认。

---

## 十三、本节练习

请完成下面练习：

1. 写出订阅 `cache:invalid` 的命令。
2. 写出发布缓存失效消息的命令。
3. 用 Go 实现 `PublishCacheInvalid`。
4. 用 Go 实现简单订阅循环。
5. 说明 Pub/Sub 为什么不可靠。
6. 判断本地缓存失效通知是否适合 Pub/Sub。
7. 判断支付成功消息是否适合 Pub/Sub。
8. 比较 Pub/Sub 和 Stream 的区别。

---

## 十四、本节小结

这一节你学习了 Pub/Sub。

你需要记住：

- Pub/Sub 是轻量发布订阅。
- 发布者 `PUBLISH`，订阅者 `SUBSCRIBE`。
- Pub/Sub 不保存消息，没有 ACK，也不能补离线消息。
- 它适合非关键通知和轻量广播。
- 重要业务消息应该使用 Stream 或专业 MQ。

下一节我们学习 Redis Stream，它比 Pub/Sub 更接近消息队列。

