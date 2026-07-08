# 4. 常用数据结构命令在 Go 中的写法

前面几阶段我们已经学习了 Redis 的 String、Hash、List、Set、ZSet。

这一节把这些命令放到 Go 代码里，看看 `go-redis/v9` 中应该怎么写。

学完这一节后，你应该能够：

- 使用 Go 操作 Redis String。
- 使用 Go 操作 Hash、List、Set、ZSet。
- 理解 `Result()`、`Err()`、`Val()` 的区别。
- 知道 Redis 命令返回值在 Go 中是什么类型。
- 写出常见业务命令的 Go 版本。

---

## 一、基础准备

假设你已经有一个 Redis client：

```go
rdb := redis.NewClient(&redis.Options{
    Addr: "127.0.0.1:6379",
})
```

每个命令都传入 context：

```go
ctx := context.Background()
```

真实 HTTP 服务中应该使用 `r.Context()`，本节示例为了简洁使用 `context.Background()`。

---

## 二、Result、Err、Val 的区别

`go-redis` 命令会返回一个命令对象。

例如：

```go
cmd := rdb.Get(ctx, "name")
```

常用取值方式：

```go
val, err := cmd.Result()
```

也可以：

```go
err := cmd.Err()
val := cmd.Val()
```

区别：

| 方法 | 含义 |
| --- | --- |
| `Result()` | 同时返回值和错误，最常用 |
| `Err()` | 只取错误 |
| `Val()` | 只取值，如果有错误可能得到零值 |

推荐大多数业务代码使用 `Result()`，避免忽略错误。

---

## 三、String：SET 和 GET

写入：

```go
err := rdb.Set(ctx, "name", "redis", 0).Err()
if err != nil {
    return err
}
```

第四个参数是过期时间。

`0` 表示不过期。

设置 5 分钟过期：

```go
err := rdb.Set(ctx, "sms:code:phone:13800138000", "888888", 5*time.Minute).Err()
```

读取：

```go
val, err := rdb.Get(ctx, "name").Result()
if err != nil {
    return err
}
fmt.Println(val)
```

缓存未命中时，`err` 会是 `redis.Nil`。

---

## 四、String：INCR 和 INCRBY

阅读量加 1：

```go
n, err := rdb.Incr(ctx, "article:view_count:1001").Result()
if err != nil {
    return err
}
fmt.Println(n)
```

增加指定数量：

```go
n, err := rdb.IncrBy(ctx, "article:view_count:1001", 10).Result()
```

返回值是自增后的结果，类型通常是 `int64`。

---

## 五、String：SET NX EX

分布式锁基础命令：

```go
ok, err := rdb.SetNX(ctx, "lock:order:1001", "request-id", 10*time.Second).Result()
if err != nil {
    return err
}
if !ok {
    return errors.New("lock already exists")
}
```

`SetNX` 返回 bool：

- `true`：设置成功。
- `false`：key 已存在。

注意：释放锁要校验 value，后面 Lua 脚本会讲。

---

## 六、Hash：HSET、HGET、HGETALL

写入用户资料：

```go
err := rdb.HSet(ctx, "user:profile:1001", map[string]any{
    "name": "tom",
    "age":  18,
    "city": "shanghai",
}).Err()
```

读取单字段：

```go
name, err := rdb.HGet(ctx, "user:profile:1001", "name").Result()
```

读取多个字段：

```go
vals, err := rdb.HMGet(ctx, "user:profile:1001", "name", "city").Result()
```

`HMGet` 返回 `[]any`，字段不存在时对应位置是 `nil`。

读取全部字段：

```go
m, err := rdb.HGetAll(ctx, "user:profile:1001").Result()
if err != nil {
    return err
}
fmt.Println(m["name"])
```

`HGetAll` 返回 `map[string]string`。

---

## 七、Hash：HINCRBY

文章统计：

```go
n, err := rdb.HIncrBy(ctx, "article:stats:1001", "view_count", 1).Result()
if err != nil {
    return err
}
fmt.Println(n)
```

适合多个计数字段放在同一个 Hash 下。

---

## 八、List：LPUSH、RPUSH、LPOP、LRANGE

写入最新动态：

```go
err := rdb.LPush(ctx, "feed:user:1001", "post:9001").Err()
```

控制长度：

```go
err := rdb.LTrim(ctx, "feed:user:1001", 0, 9).Err()
```

读取前 10 条：

```go
items, err := rdb.LRange(ctx, "feed:user:1001", 0, 9).Result()
```

简单队列：

```go
err := rdb.RPush(ctx, "task:email", "task-1").Err()
item, err := rdb.LPop(ctx, "task:email").Result()
```

如果列表为空，`LPop` 会返回 `redis.Nil`。

---

## 九、Set：SADD、SISMEMBER、SCARD

点赞用户集合：

```go
err := rdb.SAdd(ctx, "article:liked_users:1001", "user:2001").Err()
```

判断是否点赞过：

```go
ok, err := rdb.SIsMember(ctx, "article:liked_users:1001", "user:2001").Result()
if err != nil {
    return err
}
if ok {
    fmt.Println("liked")
}
```

统计点赞人数：

```go
count, err := rdb.SCard(ctx, "article:liked_users:1001").Result()
```

共同好友：

```go
friends, err := rdb.SInter(ctx, "user:friends:1001", "user:friends:1002").Result()
```

---

## 十、ZSet：ZADD、ZINCRBY、ZREVRANGE

添加排行榜成员：

```go
err := rdb.ZAdd(ctx, "rank:article:daily", redis.Z{
    Score:  100,
    Member: "article:1001",
}).Err()
```

增加分数：

```go
score, err := rdb.ZIncrBy(ctx, "rank:article:daily", 1, "article:1001").Result()
```

读取前 10 名：

```go
items, err := rdb.ZRevRangeWithScores(ctx, "rank:article:daily", 0, 9).Result()
if err != nil {
    return err
}
for _, item := range items {
    fmt.Println(item.Member, item.Score)
}
```

查看排名：

```go
rank, err := rdb.ZRevRank(ctx, "rank:article:daily", "article:1001").Result()
```

`rank` 从 0 开始。

---

## 十一、过期时间

给 key 设置过期：

```go
err := rdb.Expire(ctx, "article:detail:1001", 30*time.Minute).Err()
```

查看 TTL：

```go
ttl, err := rdb.TTL(ctx, "article:detail:1001").Result()
```

`ttl` 是 `time.Duration`。

可能返回：

```text
-1ns 表示没有过期时间
-2ns 表示 key 不存在
```

在 Go 里可以直接打印：

```go
fmt.Println(ttl)
```

---

## 十二、删除 key

删除单个 key：

```go
n, err := rdb.Del(ctx, "article:detail:1001").Result()
```

返回删除数量。

删除多个 key：

```go
n, err := rdb.Del(ctx, "a", "b", "c").Result()
```

更新数据库后删除缓存时，经常使用 `Del`。

---

## 十三、常见错误

### 1. 只用 Val 不看 Err

```go
val := rdb.Get(ctx, key).Val()
```

这样容易忽略 Redis 错误和缓存未命中。

### 2. 不处理 redis.Nil

`GET`、`HGET`、`LPOP` 等命令在数据不存在时可能返回 `redis.Nil`。

### 3. ZSet Member 类型混乱

`Member` 是 `any`，但建议统一使用字符串，避免解析混乱。

### 4. HMGet 返回 []any 不处理 nil

字段不存在时对应元素是 `nil`。

### 5. Redis 命令散落在业务代码各处

建议封装到 cache/repository 层。

---

## 十四、本节练习

请完成下面练习：

1. 使用 Go 写入和读取一个 String key。
2. 使用 `Set` 写入验证码并设置 5 分钟 TTL。
3. 使用 `Incr` 模拟文章阅读量。
4. 使用 Hash 保存用户资料并读取 `name` 字段。
5. 使用 List 保存最新 10 条动态。
6. 使用 Set 保存点赞用户并判断是否点赞。
7. 使用 ZSet 实现文章阅读排行榜。
8. 使用 `Expire` 设置过期时间。
9. 使用 `TTL` 查看剩余过期时间。
10. 所有命令都使用 `Result()` 或 `Err()` 正确处理错误。

---

## 十五、本节小结

这一节你学习了 Redis 常用数据结构命令在 Go 中的写法。

你需要记住：

- `Result()` 同时返回值和错误，最常用。
- `Err()` 只返回错误，适合只关心是否成功的命令。
- 不要只用 `Val()` 忽略错误。
- String、Hash、List、Set、ZSet 在 `go-redis` 中都有对应方法。
- 缓存未命中通常表现为 `redis.Nil`。
- Redis 命令应该封装在清晰的缓存模块里，而不是散落在 handler 中。

下一节我们会学习 JSON 序列化和反序列化，把数据库对象缓存到 Redis String 中。
