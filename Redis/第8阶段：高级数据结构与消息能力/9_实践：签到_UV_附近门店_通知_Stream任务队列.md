# 9. 实践：签到、UV、附近门店、通知、Stream 任务队列

这一节把第 8 阶段的内容串起来。

我们设计一组常见后端能力：

- 用 Bitmap 实现用户签到。
- 用 HyperLogLog 统计页面 UV。
- 用 GEO 查询附近门店。
- 用 Pub/Sub 做本地缓存失效通知。
- 用 Stream 实现可靠性更好的异步任务队列。

学完这一节后，你应该能够：

- 为不同业务选择合适 Redis 结构。
- 设计清晰的 key。
- 在 Go 中封装基础操作。
- 理解每个能力的降级和边界。
- 完成第 8 阶段实践闭环。

---

## 一、实践模块结构

推荐结构：

```text
internal/redisx/
  bitmap_sign.go
  hll_uv.go
  geo_shop.go
  pubsub_cache.go
  stream_queue.go
```

业务模块：

```text
internal/user/
internal/analytics/
internal/shop/
internal/article/
internal/task/
```

Redis 命令不要散落在 HTTP Handler 里。

---

## 二、用户签到 Bitmap

key：

```go
func UserSignKey(userID int64, month time.Time) string {
    return fmt.Sprintf("sign:user:%d:%s", userID, month.Format("200601"))
}
```

签到：

```go
func SignIn(ctx context.Context, rdb *redis.Client, userID int64, now time.Time) error {
    key := UserSignKey(userID, now)
    offset := int64(now.Day() - 1)

    return rdb.SetBit(ctx, key, offset, 1).Err()
}
```

查询：

```go
func IsSigned(ctx context.Context, rdb *redis.Client, userID int64, day time.Time) (bool, error) {
    key := UserSignKey(userID, day)
    offset := int64(day.Day() - 1)

    v, err := rdb.GetBit(ctx, key, offset).Result()
    if err != nil {
        return false, err
    }
    return v == 1, nil
}
```

统计本月签到天数：

```go
func CountSignDays(ctx context.Context, rdb *redis.Client, userID int64, month time.Time) (int64, error) {
    return rdb.BitCount(ctx, UserSignKey(userID, month), nil).Result()
}
```

---

## 三、页面 PV 和 UV

PV 使用计数器：

```text
page:pv:{page_id}:{yyyyMMdd}
```

UV 使用 HyperLogLog：

```text
page:uv:{page_id}:{yyyyMMdd}
```

Go：

```go
func RecordPageVisit(ctx context.Context, rdb *redis.Client, pageID string, visitorID string, now time.Time) error {
    date := now.Format("20060102")
    pvKey := fmt.Sprintf("page:pv:%s:%s", pageID, date)
    uvKey := fmt.Sprintf("page:uv:%s:%s", pageID, date)

    pipe := rdb.Pipeline()
    pipe.Incr(ctx, pvKey)
    pipe.PFAdd(ctx, uvKey, visitorID)
    pipe.Expire(ctx, pvKey, 90*24*time.Hour)
    pipe.Expire(ctx, uvKey, 90*24*time.Hour)
    _, err := pipe.Exec(ctx)
    return err
}
```

查询：

```go
func GetPageStats(ctx context.Context, rdb *redis.Client, pageID string, day time.Time) (int64, int64, error) {
    date := day.Format("20060102")
    pvKey := fmt.Sprintf("page:pv:%s:%s", pageID, date)
    uvKey := fmt.Sprintf("page:uv:%s:%s", pageID, date)

    pipe := rdb.Pipeline()
    pvCmd := pipe.Get(ctx, pvKey)
    uvCmd := pipe.PFCount(ctx, uvKey)
    _, err := pipe.Exec(ctx)
    if err != nil && !errors.Is(err, redis.Nil) {
        return 0, 0, err
    }

    pv, _ := pvCmd.Int64()
    uv, _ := uvCmd.Result()
    return pv, uv, nil
}
```

UV 是估算值，不用于发奖或计费。

---

## 四、附近门店 GEO

GEO key：

```text
shop:geo:{city}
```

添加门店：

```go
func AddShop(ctx context.Context, rdb *redis.Client, city string, shopID int64, longitude, latitude float64) error {
    key := fmt.Sprintf("shop:geo:%s", city)
    return rdb.GeoAdd(ctx, key, &redis.GeoLocation{
        Name:      fmt.Sprintf("shop:%d", shopID),
        Longitude: longitude,
        Latitude:  latitude,
    }).Err()
}
```

查询附近：

```go
func NearbyShops(ctx context.Context, rdb *redis.Client, city string, longitude, latitude float64) ([]redis.GeoLocation, error) {
    key := fmt.Sprintf("shop:geo:%s", city)
    return rdb.GeoSearchLocation(ctx, key, &redis.GeoSearchLocationQuery{
        GeoSearchQuery: redis.GeoSearchQuery{
            Longitude:  longitude,
            Latitude:   latitude,
            Radius:     3,
            RadiusUnit: "km",
            Sort:       "ASC",
            Count:      20,
        },
        WithDist: true,
    }).Result()
}
```

查询到 `shop_id` 后，再批量查门店详情缓存。

---

## 五、本地缓存失效 Pub/Sub

频道：

```text
cache:invalid
```

发布：

```go
func PublishInvalid(ctx context.Context, rdb *redis.Client, cacheKey string) error {
    return rdb.Publish(ctx, "cache:invalid", cacheKey).Err()
}
```

订阅：

```go
func RunInvalidSubscriber(ctx context.Context, rdb *redis.Client, localCache LocalCache) error {
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
            localCache.Delete(msg.Payload)
        }
    }
}
```

本地缓存必须有 TTL。

Pub/Sub 消息丢失时，TTL 是兜底。

---

## 六、Stream 任务队列

Stream key：

```text
task:email
```

写入任务：

```go
func AddEmailTask(ctx context.Context, rdb *redis.Client, userID int64, template string) (string, error) {
    return rdb.XAdd(ctx, &redis.XAddArgs{
        Stream:       "task:email",
        MaxLenApprox: 100000,
        Values: map[string]interface{}{
            "user_id":  userID,
            "template": template,
        },
    }).Result()
}
```

创建消费者组：

```go
func EnsureEmailGroup(ctx context.Context, rdb *redis.Client) error {
    err := rdb.XGroupCreateMkStream(ctx, "task:email", "email-workers", "0").Err()
    if err != nil && !strings.Contains(err.Error(), "BUSYGROUP") {
        return err
    }
    return nil
}
```

---

## 七、Stream worker

```go
func RunEmailWorker(ctx context.Context, rdb *redis.Client, consumer string, handler func(context.Context, redis.XMessage) error) error {
    if err := EnsureEmailGroup(ctx, rdb); err != nil {
        return err
    }

    for {
        streams, err := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
            Group:    "email-workers",
            Consumer: consumer,
            Streams:  []string{"task:email", ">"},
            Count:    10,
            Block:    5 * time.Second,
        }).Result()
        if err != nil {
            if errors.Is(err, redis.Nil) {
                continue
            }
            log.Printf("read email task failed err=%v", err)
            continue
        }

        for _, stream := range streams {
            for _, msg := range stream.Messages {
                if err := handler(ctx, msg); err != nil {
                    log.Printf("handle email task failed id=%s err=%v", msg.ID, err)
                    continue
                }

                if err := rdb.XAck(ctx, "task:email", "email-workers", msg.ID).Err(); err != nil {
                    log.Printf("ack email task failed id=%s err=%v", msg.ID, err)
                }
            }
        }
    }
}
```

handler 必须幂等。

例如同一封邮件任务不能无限重复发送，可以用任务表或发送记录去重。

---

## 八、Redis key 汇总

```text
sign:user:{user_id}:{yyyyMM}
page:pv:{page_id}:{yyyyMMdd}
page:uv:{page_id}:{yyyyMMdd}
shop:geo:{city}
cache:invalid
task:email
```

这些 key 表达了：

- 业务。
- 结构。
- 对象。
- 时间或区域。

排查时更容易定位。

---

## 九、降级和边界

| 能力 | 边界 |
| --- | --- |
| Bitmap 签到 | offset 规则必须稳定 |
| HyperLogLog UV | 有误差，不能取成员 |
| GEO 附近门店 | 不适合复杂 GIS |
| Pub/Sub 通知 | 消息可能丢 |
| Stream 队列 | 需要 ACK、Pending 处理和幂等 |

Redis 很强，但每个能力都有边界。

设计时要把边界写进代码和文档。

---

## 十、监控指标

建议关注：

- 签到写入失败次数。
- PV/UV 写入失败次数。
- GEO 查询耗时。
- Pub/Sub 发布失败次数。
- 本地缓存命中率。
- Stream 长度。
- Stream Pending 数量。
- 消息处理失败次数。
- ACK 失败次数。

Stream 相关指标尤其重要。

Pending 增长通常意味着消费者处理异常。

---

## 十一、常见错误

### 1. UV 统计用于精确结算

HyperLogLog 有误差，不适合结算。

### 2. Pub/Sub 消息没有兜底

本地缓存失效通知丢失后，可能长期读旧数据。

### 3. Stream 消费不 ACK

Pending 会不断积压。

### 4. Stream 消费不幂等

重复投递会造成重复发送、重复更新。

### 5. GEO 查询后不查详情状态

门店可能已关闭或不可用。

---

## 十二、本节练习

请完成下面练习：

1. 实现用户签到和查询。
2. 统计用户本月签到天数。
3. 实现页面 PV/UV 统计。
4. 实现附近门店查询。
5. 实现缓存失效发布和订阅。
6. 实现 Stream 邮件任务队列。
7. 为 Stream worker 增加 ACK。
8. 设计 Pending 消息重试策略。
9. 为每个能力写出降级策略。

---

## 十三、本节小结

这一节完成了第 8 阶段的实践闭环。

你需要记住：

- Bitmap 适合签到和布尔状态。
- HyperLogLog 适合允许误差的 UV 统计。
- GEO 适合附近门店这类轻量位置查询。
- Pub/Sub 适合轻量通知，但消息可能丢。
- Stream 适合可靠性更好的轻量异步任务。
- Stream 消费必须 ACK、监控 Pending，并保证业务幂等。

学完第 8 阶段，你已经掌握了 Redis 的扩展数据结构和消息能力。下一阶段会进入持久化、过期与内存管理，理解 Redis 数据如何保存、清理，以及内存不足时会发生什么。

