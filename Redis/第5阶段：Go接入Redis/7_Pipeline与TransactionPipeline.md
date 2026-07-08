# 7. Pipeline 与 Transaction Pipeline

在 Go 服务中访问 Redis 时，网络往返 RTT 是重要成本。

如果一次请求要执行多条 Redis 命令，逐条发送会产生多次网络往返。Pipeline 可以把多条命令一次性发送给 Redis，减少网络开销。

学完这一节后，你应该能够：

- 理解 Pipeline 解决什么问题。
- 使用 `Pipeline` 批量执行 Redis 命令。
- 获取 Pipeline 中每条命令的结果。
- 理解 Pipeline 不是事务。
- 使用 `TxPipeline` 发送 `MULTI/EXEC` 包裹的命令。
- 知道 Pipeline 和 Transaction Pipeline 的适用场景与局限。

---

## 一、为什么需要 Pipeline

假设接口要读取 3 篇文章详情：

```go
rdb.Get(ctx, "article:detail:1001")
rdb.Get(ctx, "article:detail:1002")
rdb.Get(ctx, "article:detail:1003")
```

如果逐条执行，客户端和 Redis 之间有 3 次请求响应。

Pipeline 的思路是：

```text
把多条命令一次性发给 Redis
Redis 依次执行
一次性把结果返回给客户端
```

这样可以减少网络往返，提高批量操作效率。

---

## 二、Pipeline 不是事务

先记住一句话：

```text
Pipeline 只是批量发送命令，不保证事务原子性。
```

也就是说，Pipeline 中的命令仍然是多条命令。

它们会被 Redis 执行，但 Pipeline 本身不代表“要么全部成功，要么全部失败”。

如果你需要事务语义，可以看后面的 `TxPipeline`。

如果你需要复杂条件判断和原子更新，通常要用 Lua 脚本。

---

## 三、Pipeline 基本写法

示例：批量写入文章标题。

```go
pipe := rdb.Pipeline()

cmd1 := pipe.Set(ctx, "article:title:1001", "Redis 入门", 30*time.Minute)
cmd2 := pipe.Set(ctx, "article:title:1002", "缓存设计", 30*time.Minute)
cmd3 := pipe.Set(ctx, "article:title:1003", "Go 接入 Redis", 30*time.Minute)

_, err := pipe.Exec(ctx)
if err != nil {
    return err
}

fmt.Println(cmd1.Err(), cmd2.Err(), cmd3.Err())
```

流程：

```text
创建 Pipeline
把命令放进 Pipeline
Exec 一次性执行
从命令对象读取结果
```

---

## 四、Pipeline 读取结果

批量读取：

```go
pipe := rdb.Pipeline()

cmd1 := pipe.Get(ctx, "article:detail:1001")
cmd2 := pipe.Get(ctx, "article:detail:1002")
cmd3 := pipe.Get(ctx, "article:detail:1003")

_, err := pipe.Exec(ctx)
if err != nil && !errors.Is(err, redis.Nil) {
    return err
}

v1, err1 := cmd1.Result()
v2, err2 := cmd2.Result()
v3, err3 := cmd3.Result()
```

注意：某个 `GET` 未命中会返回 `redis.Nil`。

Pipeline 的 `Exec` 也可能返回其中某条命令的错误。

所以批量读取时，通常要逐个命令检查结果。

---

## 五、使用 Pipelined 简化写法

`go-redis` 也提供 `Pipelined`：

```go
cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
    pipe.Set(ctx, "a", "1", time.Minute)
    pipe.Set(ctx, "b", "2", time.Minute)
    pipe.Set(ctx, "c", "3", time.Minute)
    return nil
})
if err != nil {
    return err
}

fmt.Println(cmds)
```

如果你需要拿每条命令的具体类型结果，保存命令对象更清晰：

```go
var getA *redis.StringCmd
_, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
    getA = pipe.Get(ctx, "a")
    return nil
})
val, err := getA.Result()
```

---

## 六、场景一：批量读取缓存

假设文章列表页要读取多个文章详情缓存。

```go
func GetArticleCaches(ctx context.Context, rdb *redis.Client, ids []int64) map[int64]string {
    pipe := rdb.Pipeline()
    cmds := make(map[int64]*redis.StringCmd, len(ids))

    for _, id := range ids {
        key := fmt.Sprintf("article:detail:%d", id)
        cmds[id] = pipe.Get(ctx, key)
    }

    _, _ = pipe.Exec(ctx)

    result := make(map[int64]string)
    for id, cmd := range cmds {
        val, err := cmd.Result()
        if errors.Is(err, redis.Nil) {
            continue
        }
        if err != nil {
            continue
        }
        result[id] = val
    }

    return result
}
```

真实项目里不要直接忽略错误，至少要记录日志。

这里重点是：Pipeline 可以一次发送多个 `GET`。

---

## 七、场景二：写缓存并设置多个相关 key

比如文章详情写缓存，同时写标题缓存和统计缓存：

```go
pipe := rdb.Pipeline()
pipe.Set(ctx, "article:detail:1001", articleJSON, 30*time.Minute)
pipe.Set(ctx, "article:title:1001", title, 30*time.Minute)
pipe.HSet(ctx, "article:stats:1001", "view_count", 0, "like_count", 0)
_, err := pipe.Exec(ctx)
```

这些命令会一起发给 Redis，减少 RTT。

但注意：这不是事务。

如果某条命令失败，其他命令可能已经执行。

---

## 八、Pipeline 的批量大小

Pipeline 不是越大越好。

一次塞太多命令可能导致：

- 客户端内存增加。
- Redis 单次处理时间变长。
- 响应包过大。
- 某个请求占用 Redis 太久。

比如一次 Pipeline 20 条命令通常没问题。

一次 Pipeline 10 万条命令就很危险。

真实项目里要控制批量大小，例如分批处理。

---

## 九、TxPipeline 是什么

`TxPipeline` 会使用 Redis 的 `MULTI/EXEC` 包裹命令。

示例：

```go
pipe := rdb.TxPipeline()
pipe.Set(ctx, "a", "1", time.Minute)
pipe.Set(ctx, "b", "2", time.Minute)
_, err := pipe.Exec(ctx)
```

大致对应 Redis：

```redis
MULTI
SET a 1 EX 60
SET b 2 EX 60
EXEC
```

它能保证命令在 `EXEC` 时按顺序执行，中间不会插入其他客户端命令。

但 Redis 事务和关系型数据库事务不同。

---

## 十、Redis 事务的边界

Redis `MULTI/EXEC` 不是 MySQL 那种完整事务。

你需要知道：

- 它不会自动回滚已经执行的命令。
- 它不适合复杂条件判断。
- 命令执行错误需要自己处理。
- 读后判断再写的复杂逻辑通常不适合只用 TxPipeline。

比如扣库存：

```text
先 GET stock
判断 stock > 0
再 DECR
```

如果要保证这个逻辑原子，Lua 脚本更合适。

---

## 十一、TxPipeline 适合什么

适合：

- 多条命令希望连续执行，中间不插入其他命令。
- 命令之间没有复杂条件判断。
- 可以接受 Redis 事务语义。

例如：

```go
pipe := rdb.TxPipeline()
pipe.Incr(ctx, "article:view_count:1001")
pipe.ZIncrBy(ctx, "rank:article:daily", 1, "article:1001")
_, err := pipe.Exec(ctx)
```

表示阅读量计数和排行榜分数都增加。

但如果其中一条命令失败，仍要检查错误并设计补偿。

---

## 十二、Pipeline 与 MGET/MSET 的区别

`MGET` 是 Redis 命令：

```redis
MGET key1 key2 key3
```

Pipeline 是客户端批量发送多条命令：

```text
GET key1
TTL key1
GET key2
TTL key2
```

如果只是批量读 String，`MGET` 很方便。

如果要执行不同类型命令，Pipeline 更灵活。

---

## 十三、常见错误

### 1. 以为 Pipeline 是事务

Pipeline 不保证原子性。

### 2. Exec 后不检查单条命令结果

某些命令可能返回 `redis.Nil` 或其他错误。

### 3. Pipeline 太大

大批量命令要分批执行。

### 4. 用 TxPipeline 处理复杂条件逻辑

复杂原子逻辑通常应该用 Lua。

### 5. 忽略上下文超时

Pipeline 命令多，更要关注 `ctx` 超时。

---

## 十四、本节练习

请完成下面练习：

1. 使用 Pipeline 批量 `SET` 3 个 key。
2. 使用 Pipeline 批量 `GET` 3 个 key。
3. 让其中一个 key 不存在，观察 `redis.Nil`。
4. 使用 Pipeline 同时执行 `GET` 和 `TTL`。
5. 使用 Pipeline 批量写入 10 篇文章缓存。
6. 思考 Pipeline 为什么能减少 RTT。
7. 使用 TxPipeline 同时执行 `INCR` 和 `ZINCRBY`。
8. 思考 TxPipeline 和 MySQL 事务有什么不同。
9. 思考扣库存为什么更适合 Lua。
10. 设计一个 Pipeline 批量读取文章列表缓存的函数。

---

## 十五、本节小结

这一节你学习了 Pipeline 和 Transaction Pipeline。

你需要记住：

- Pipeline 用来批量发送命令，减少网络往返。
- Pipeline 不是事务，不保证全部命令原子成功。
- `Exec` 后仍然要检查每条命令结果。
- Pipeline 批量大小要控制。
- `TxPipeline` 使用 `MULTI/EXEC`，但不是关系型数据库事务。
- 复杂条件判断和原子更新通常更适合 Lua 脚本。

下一节我们学习 Lua 脚本：`Eval`、`EvalSha`，以及如何用 Lua 实现释放锁和扣库存这类原子逻辑。
