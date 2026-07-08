# 9. 可选增强：热榜、Stream 访问日志、Bitmap、本地缓存

这一节讨论短链接项目的可选增强。

主链路完成后，再考虑这些能力。不要在项目一开始就把所有高级结构都堆进去。

学完这一节后，你应该能够：

- 使用 ZSet 设计热门短链接排行榜。
- 使用 Stream 异步记录访问日志。
- 使用 Bitmap 做每日访问标记。
- 理解本地缓存和多级缓存。
- 判断哪些增强适合当前业务。

---

## 一、增强能力概览

| 能力 | Redis 结构 | 用途 |
| --- | --- | --- |
| 热门链接榜 | ZSet | 按访问次数排名 |
| 访问日志 | Stream | 异步记录访问事件 |
| 每日访问标记 | Bitmap | 统计某天是否访问 |
| 本地缓存 | 进程内缓存 | 降低 Redis 压力 |
| 严格限流 | Lua + ZSet | 滑动窗口限流 |

这些都不是第一版必须项。

项目第一版要先保证：

- 创建能用。
- 跳转稳定。
- 缓存正确。
- 限流有效。
- 更新后缓存能失效。

---

## 二、ZSet 热门链接榜

热门链接可以用 ZSet。

key：

```text
shortlink:hot:daily:20260705
```

每次跳转：

```redis
ZINCRBY shortlink:hot:daily:20260705 1 aB9xK2
```

读取 Top 10：

```redis
ZREVRANGE shortlink:hot:daily:20260705 0 9 WITHSCORES
```

Go 示例：

```go
func (c *ShortLinkCache) IncrHotRank(ctx context.Context, code string, day string) error {
    key := "shortlink:hot:daily:" + day
    return c.rdb.ZIncrBy(ctx, key, 1, code).Err()
}
```

排行榜 key 要设置过期时间。

例如保留 30 天。

---

## 三、Stream 访问日志

如果你想记录每次访问事件，可以用 Stream。

key：

```text
shortlink:visit_stream
```

写入事件：

```redis
XADD shortlink:visit_stream * code aB9xK2 user_agent Chrome ip 127.0.0.1
```

Go 示例：

```go
func (c *ShortLinkCache) AddVisitEvent(ctx context.Context, code string, ip string, ua string) error {
    return c.rdb.XAdd(ctx, &redis.XAddArgs{
        Stream: "shortlink:visit_stream",
        Values: map[string]any{
            "code": code,
            "ip":   ip,
            "ua":   ua,
        },
    }).Err()
}
```

消费端可以异步聚合：

- 访问次数。
- 独立 IP。
- 浏览器分布。
- 地域统计。

---

## 四、Stream 使用注意

Stream 不是无限日志仓库。

要考虑：

- 消费组。
- 消息确认。
- 重试和死信。
- 最大长度裁剪。
- 消费延迟监控。

可以限制长度：

```go
c.rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "shortlink:visit_stream",
    MaxLenApprox: 100000,
    Values: values,
})
```

如果访问量很大，专业日志系统可能更合适。

---

## 五、Bitmap 每日访问标记

Bitmap 适合记录某个编号是否访问过。

如果每个短链接有数字 ID，可以设计：

```text
shortlink:visited:20260705
```

访问时：

```redis
SETBIT shortlink:visited:20260705 {link_id} 1
```

统计当天被访问过的短链接数量：

```redis
BITCOUNT shortlink:visited:20260705
```

Go 示例：

```go
func (c *ShortLinkCache) MarkVisited(ctx context.Context, linkID int64, day string) error {
    key := "shortlink:visited:" + day
    return c.rdb.SetBit(ctx, key, linkID, 1).Err()
}
```

Bitmap 更适合 ID 稠密、范围可控的场景。

---

## 六、本地缓存

对于极热短链接，可以加入本地缓存。

请求链路：

```text
本地缓存 -> Redis -> 数据库
```

优点：

- 降低 Redis 压力。
- 进一步减少网络延迟。

风险：

- 多实例缓存不一致。
- 更新后失效更复杂。
- 内存占用需要控制。

适合缓存：

- 极热短链接详情。
- TTL 很短的数据。
- 可以接受短时间不一致的数据。

---

## 七、多级缓存失效

如果用了本地缓存，更新短链接时要考虑：

```text
删除数据库对应 Redis 缓存
删除本地缓存
通知其他实例删除本地缓存
```

通知方式：

- Redis Pub/Sub。
- 消息队列。
- 配置中心事件。
- 短 TTL 自然过期。

学习项目可以只使用短 TTL。

不要一开始就把多级缓存做得很重。

---

## 八、Lua + ZSet 滑动窗口限流

如果创建接口需要更平滑的限流，可以使用 ZSet 做滑动窗口。

思路：

```text
移除窗口外记录
统计窗口内请求数
未超过限制则写入当前请求时间
设置 key 过期
```

适合：

- 登录接口。
- 短信接口。
- 高价值写接口。

短链接创建接口如果只是普通业务，固定窗口通常够用。

---

## 九、如何选择增强项

选择标准：

| 问题 | 可选方案 |
| --- | --- |
| 想展示热门链接 | ZSet |
| 想保留访问明细 | Stream |
| 想知道每天哪些链接被访问 | Bitmap |
| Redis 压力太大 | 本地缓存 |
| 限流边界不够精确 | Lua + ZSet |

每个增强项都要回答：

```text
它解决了什么明确问题？
维护成本是否值得？
故障时如何降级？
数据是否需要落库？
```

---

## 十、常见错误

### 1. 主链路没完成就做增强

项目会变得复杂但不可用。

### 2. Stream 不裁剪

消息会持续增长。

### 3. 本地缓存没有失效策略

更新后各实例读到不同数据。

### 4. Bitmap 用在稀疏巨大 ID 上

空间可能浪费。

### 5. 热榜 key 不设置过期

历史排行榜会越积越多。

---

## 十一、本节练习

请完成下面练习：

1. 用 ZSet 实现每日热门链接 Top 10。
2. 给热榜 key 设置 30 天 TTL。
3. 用 Stream 写入访问事件。
4. 设计一个访问日志消费者。
5. 用 Bitmap 标记当天被访问过的链接。
6. 写出本地缓存失效方案。

---

## 十二、本节小结

这一节介绍了短链接项目的可选增强。

你需要记住：

- ZSet 适合排行榜。
- Stream 适合异步访问日志。
- Bitmap 适合每日访问标记。
- 本地缓存能降低 Redis 压力，但一致性更复杂。
- 增强项必须服务明确问题。

下一节我们用测试清单、故障降级和项目复盘完成第 13 阶段。

