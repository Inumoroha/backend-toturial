# 8. 统计接口：Pipeline 读取计数与详情

这一节实现统计接口。

统计接口通常需要同时读取短链接详情和访问计数。Redis Pipeline 可以把多个命令一次发给 Redis，减少网络往返。

学完这一节后，你应该能够：

- 实现 `GET /links/:code/stats`。
- 使用 Redis 读取访问计数。
- 使用 Pipeline 合并多个 Redis 命令。
- 处理计数不存在的情况。
- 理解统计数据的可靠性边界。

---

## 一、统计接口目标

接口：

```text
GET /links/:code/stats
```

响应：

```json
{
  "code": "aB9xK2",
  "original_url": "https://example.com/articles/100",
  "visit_count": 12345
}
```

数据来源：

| 字段 | 来源 |
| --- | --- |
| `code` | 路径参数 |
| `original_url` | 数据库或详情缓存 |
| `visit_count` | Redis 计数器 |

---

## 二、读取访问计数

Redis key：

```text
shortlink:visit_count:{code}
```

命令：

```redis
GET shortlink:visit_count:aB9xK2
```

go-redis 示例：

```go
func (c *ShortLinkCache) GetVisitCount(ctx context.Context, code string) (int64, error) {
    n, err := c.rdb.Get(ctx, visitCountKey(code)).Int64()
    if errors.Is(err, redis.Nil) {
        return 0, nil
    }
    if err != nil {
        return 0, err
    }
    return n, nil
}
```

计数 key 不存在时返回 0。

---

## 三、为什么用 Pipeline

如果统计接口要读多个 Redis key：

```text
GET shortlink:detail:{code}
GET shortlink:visit_count:{code}
EXISTS shortlink:not_found:{code}
```

普通方式需要多次网络往返。

Pipeline 可以一次发送多个命令：

```text
客户端 -> Redis: GET + GET + EXISTS
Redis -> 客户端: 结果1 + 结果2 + 结果3
```

Pipeline 不保证事务原子性。

它主要优化网络往返。

---

## 四、Pipeline 示例

示例：

```go
func (c *ShortLinkCache) GetStatsCache(ctx context.Context, code string) (*CachedShortLink, int64, error) {
    pipe := c.rdb.Pipeline()

    detailCmd := pipe.Get(ctx, detailKey(code))
    countCmd := pipe.Get(ctx, visitCountKey(code))

    _, err := pipe.Exec(ctx)
    if err != nil && !errors.Is(err, redis.Nil) {
        return nil, 0, err
    }

    var link *CachedShortLink
    if detailCmd.Err() == nil {
        var cached CachedShortLink
        if err := json.Unmarshal([]byte(detailCmd.Val()), &cached); err != nil {
            return nil, 0, err
        }
        link = &cached
    }

    var count int64
    if countCmd.Err() == nil {
        count, _ = countCmd.Int64()
    }

    return link, count, nil
}
```

注意：

- `pipe.Exec` 返回 `redis.Nil` 时不一定是整体失败。
- 每个命令要单独检查结果。
- JSON 解析失败要记录。

---

## 五、服务层统计逻辑

伪代码：

```go
func (s *ShortLinkService) GetStats(ctx context.Context, code string) (*LinkStats, error) {
    cached, count, err := s.cache.GetStatsCache(ctx, code)
    if err != nil {
        log.Printf("get stats cache failed: code=%s err=%v", code, err)
    }

    var originalURL string
    if cached != nil {
        originalURL = cached.OriginalURL
    } else {
        link, err := s.repo.FindByCode(ctx, code)
        if err != nil {
            return nil, err
        }
        originalURL = link.OriginalURL
    }

    return &LinkStats{
        Code:        code,
        OriginalURL: originalURL,
        VisitCount:  count,
    }, nil
}
```

如果 Redis 故障，计数可以返回 0 或提示统计暂不可用。

根据产品要求选择。

---

## 六、计数同步到数据库

如果访问次数很重要，不能只放 Redis。

可以设计数据库字段：

```sql
ALTER TABLE short_links ADD COLUMN visit_count BIGINT NOT NULL DEFAULT 0;
```

然后定时同步：

```text
每 1 分钟扫描活跃 code
读取 Redis 计数增量
累加到数据库
清理或保留 Redis 计数
```

更好的方案是记录访问事件，再异步聚合。

本项目先把 Redis 计数作为实时统计。

---

## 七、Pipeline 和事务的区别

Pipeline：

```text
减少网络往返
不保证多个命令原子执行
```

事务：

```text
MULTI / EXEC
多个命令按顺序执行
可用于需要事务语义的场景
```

统计读取不需要事务。

使用 Pipeline 更合适。

---

## 八、统计接口降级

Redis 失败时可选策略：

| 场景 | 策略 |
| --- | --- |
| 详情缓存失败 | 查数据库 |
| 计数读取失败 | 返回 0 或返回统计暂不可用 |
| Pipeline 超时 | 记录日志，查数据库详情 |
| 反序列化失败 | 删除坏缓存，查数据库 |

如果统计用于后台管理，可以返回部分数据：

```json
{
  "code": "aB9xK2",
  "original_url": "https://example.com/articles/100",
  "visit_count": null,
  "stats_available": false
}
```

---

## 九、常见错误

### 1. 计数不存在时返回 500

新短链接访问次数为 0 是正常情况。

### 2. Pipeline 当事务用

Pipeline 不保证原子性。

### 3. 统计接口强依赖缓存详情

详情缓存没有命中时可以查数据库。

### 4. 忽略单个命令错误

Pipeline 执行后仍要看每个命令的结果。

### 5. 把实时计数当绝对准确

Redis 计数如果没有落库，就可能丢。

---

## 十、本节练习

请完成下面练习：

1. 实现 `GetVisitCount`。
2. 实现 `GET /links/:code/stats`。
3. 使用 Pipeline 同时读取详情缓存和访问计数。
4. 处理计数 key 不存在的情况。
5. 模拟 Redis 故障，设计统计接口降级返回。
6. 思考访问计数是否需要同步到数据库。

---

## 十一、本节小结

这一节完成了统计接口。

你需要记住：

- 访问计数可以使用 Redis `INCR` 和 `GET`。
- 计数不存在时就是 0。
- Pipeline 用来减少网络往返，不是事务。
- 统计接口可以接受部分降级。
- 重要统计最终要考虑落库或异步聚合。

下一节我们讨论可选增强：热榜、Stream 访问日志、Bitmap 和本地缓存。

