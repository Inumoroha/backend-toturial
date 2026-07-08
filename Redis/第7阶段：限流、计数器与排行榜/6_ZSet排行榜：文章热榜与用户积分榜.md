# 6. ZSet 排行榜：文章热榜与用户积分榜

排行榜是 Redis ZSet 最典型的应用。

ZSet 既能保存成员，又能按分数排序，非常适合文章热榜、用户积分榜、商品销量榜和游戏分数榜。

学完这一节后，你应该能够：

- 使用 ZSet 实现排行榜。
- 查询 Top N 和用户排名。
- 给成员增加分数。
- 设计日榜、周榜、总榜 key。
- 理解排行榜的时间维度、清理和一致性。

---

## 一、ZSet 为什么适合排行榜

ZSet 每个 member 对应一个 score：

```text
member = 文章 ID
score = 热度分数
```

写入：

```redis
ZADD rank:article:daily:20260705 100 1001
```

增加分数：

```redis
ZINCRBY rank:article:daily:20260705 1 1001
```

查询前 10：

```redis
ZREVRANGE rank:article:daily:20260705 0 9 WITHSCORES
```

查询排名：

```redis
ZREVRANK rank:article:daily:20260705 1001
```

---

## 二、为什么用 ZREVRANGE

排行榜通常分数越高排名越靠前。

`ZRANGE` 默认从小到大。

所以常用：

```redis
ZREVRANGE key 0 9 WITHSCORES
```

表示按 score 从大到小取前 10。

Redis 新版本也可以用：

```redis
ZRANGE key 0 9 REV WITHSCORES
```

本教程继续使用容易理解的 `ZREVRANGE`。

---

## 三、文章日榜 key 设计

按天分榜：

```text
rank:article:daily:20260705
```

按周分榜：

```text
rank:article:weekly:2026W27
```

总榜：

```text
rank:article:all
```

不要把所有时间维度都混在一个 key 里。

日榜可以设置 TTL：

```redis
EXPIRE rank:article:daily:20260705 2592000
```

例如保留 30 天。

---

## 四、阅读量驱动排行榜

用户阅读文章时：

```redis
ZINCRBY rank:article:daily:20260705 1 1001
ZINCRBY rank:article:weekly:2026W27 1 1001
ZINCRBY rank:article:all 1 1001
```

如果还要保存阅读计数：

```redis
INCR counter:article:view:1001
```

这几步可以 Pipeline 批量发送。

如果希望 Redis 内部原子，可以用 Lua。

大多数排行榜允许短暂不一致，Pipeline 通常够用。

---

## 五、Go 更新文章热榜

```go
func IncrArticleRank(ctx context.Context, rdb *redis.Client, articleID int64, now time.Time) error {
    dailyKey := fmt.Sprintf("rank:article:daily:%s", now.Format("20060102"))
    weeklyKey := fmt.Sprintf("rank:article:weekly:%dW%02d", now.Year(), weekOfYear(now))
    allKey := "rank:article:all"

    member := strconv.FormatInt(articleID, 10)

    pipe := rdb.Pipeline()
    pipe.ZIncrBy(ctx, dailyKey, 1, member)
    pipe.ZIncrBy(ctx, weeklyKey, 1, member)
    pipe.ZIncrBy(ctx, allKey, 1, member)
    pipe.Expire(ctx, dailyKey, 30*24*time.Hour)
    pipe.Expire(ctx, weeklyKey, 90*24*time.Hour)
    _, err := pipe.Exec(ctx)
    return err
}
```

`weekOfYear` 可以用标准库的 `ISOWeek()` 封装。

---

## 六、查询 Top N

Go 示例：

```go
func GetArticleTopN(ctx context.Context, rdb *redis.Client, key string, n int64) ([]redis.Z, error) {
    return rdb.ZRevRangeWithScores(ctx, key, 0, n-1).Result()
}
```

返回的 `redis.Z`：

```go
type Z struct {
    Score  float64
    Member interface{}
}
```

业务层通常还要根据 articleID 批量查询文章详情。

不要把完整文章内容塞进 ZSet member。

member 放 ID，详情从缓存或数据库查。

---

## 七、查询某篇文章排名

```go
func GetArticleRank(ctx context.Context, rdb *redis.Client, key string, articleID int64) (int64, error) {
    member := strconv.FormatInt(articleID, 10)

    rank, err := rdb.ZRevRank(ctx, key, member).Result()
    if err != nil {
        return 0, err
    }

    return rank + 1, nil
}
```

`ZREVRANK` 返回从 0 开始的排名。

展示给用户时通常加 1。

如果 member 不存在，go-redis 会返回 `redis.Nil`。

---

## 八、用户积分榜

用户积分榜：

```text
rank:user:score
```

增加积分：

```redis
ZINCRBY rank:user:score 10 1001
```

查询前 100：

```redis
ZREVRANGE rank:user:score 0 99 WITHSCORES
```

查询用户排名：

```redis
ZREVRANK rank:user:score 1001
```

积分榜通常更重要，要考虑数据库持久化。

Redis 可以做实时榜，数据库保存最终积分流水。

---

## 九、分数相同怎么排序

ZSet 的 score 相同，Redis 会按 member 字典序排序。

这可能不是你想要的业务顺序。

如果要按“分数高优先，时间早优先”，可以把时间因素编码进 score。

例如：

```text
score = points * 1_000_000_000 + time_weight
```

但这会让分数语义变复杂。

更常见的做法是：

- 榜单只按主分数排序。
- 详情页再展示并列。
- 对严格排序需求交给数据库或专门排序服务。

---

## 十、热度分数不一定等于阅读量

文章热榜可以综合多个因素：

```text
热度 = 阅读 * 1 + 点赞 * 5 + 评论 * 10 + 收藏 * 8
```

对应更新：

```redis
ZINCRBY rank:article:daily:20260705 1 1001     # 阅读
ZINCRBY rank:article:daily:20260705 5 1001     # 点赞
ZINCRBY rank:article:daily:20260705 10 1001    # 评论
```

要注意刷量和风控。

热度分数越影响曝光，越容易被攻击。

---

## 十一、排行榜大小控制

如果榜单成员无限增长，ZSet 会越来越大。

可以定期保留 Top N：

```redis
ZREMRANGEBYRANK rank:article:daily:20260705 0 -10001
```

这条命令含义是删除排名较低的成员，只保留前 10000 左右。

使用时要先确认排序方向。

如果不熟悉，建议写脚本或后台任务谨慎处理。

---

## 十二、排行榜和详情缓存

排行榜只保存 ID 和分数。

接口返回榜单时：

```text
1. 从 ZSet 取 Top N article_id。
2. 批量 MGET 文章详情缓存。
3. 缓存未命中再查数据库。
4. 按 ZSet 返回顺序组装结果。
```

不要在 ZSet member 里存 JSON。

否则更新文章标题、作者等信息会很麻烦。

---

## 十三、常见错误

### 1. member 存完整对象

对象变化后难以维护，ZSet 也会变大。

### 2. 榜单没有时间维度

日榜、周榜、总榜混在一起，难以解释。

### 3. 不设置历史榜单 TTL

旧榜单一直占内存。

### 4. 不处理 `redis.Nil`

查询不存在成员排名时会报缓存未命中。

### 5. 忽视刷榜

排行榜一旦影响流量分发，就必须考虑风控。

---

## 十四、本节练习

请完成下面练习：

1. 用 ZSet 设计文章日榜 key。
2. 写出给文章阅读量加分的命令。
3. 查询日榜 Top 10。
4. 查询某篇文章的排名。
5. 用 Go 封装 `IncrArticleRank`。
6. 设计用户积分榜 key。
7. 思考分数相同时如何展示。
8. 设计排行榜详情查询流程。

---

## 十五、本节小结

这一节你学习了 ZSet 排行榜。

你需要记住：

- ZSet 的 member 保存对象 ID，score 保存分数。
- `ZINCRBY` 用于加分，`ZREVRANGE` 用于查询 Top N。
- 排行榜要设计日榜、周榜、总榜等时间维度。
- 榜单只保存 ID，详情从缓存或数据库读取。
- 排行榜要考虑大小控制、历史清理和刷榜。

下一节我们学习 ZSet 的另一个经典用法：延迟队列。

