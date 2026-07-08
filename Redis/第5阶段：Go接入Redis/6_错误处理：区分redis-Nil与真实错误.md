# 6. 错误处理：区分 redis.Nil 与真实错误

Go 接入 Redis 时，错误处理非常重要。

很多缓存 bug 不是命令不会写，而是把“缓存未命中”和“Redis 故障”混为一谈：该回源数据库时返回了 500，该报警时却把错误吞掉了。

学完这一节后，你应该能够：

- 理解 `redis.Nil` 表示缓存未命中。
- 区分缓存未命中、Redis 系统错误、JSON 错误、context 超时。
- 为缓存读取设计清晰的错误处理分支。
- 知道什么时候降级到数据库，什么时候返回错误。
- 避免把所有 Redis 错误都吞掉。

---

## 一、redis.Nil 是什么

当你执行：

```go
val, err := rdb.Get(ctx, key).Result()
```

如果 key 不存在，`err` 通常是：

```go
redis.Nil
```

这不是 Redis 挂了，也不是网络错误。

它表示：

```text
缓存未命中
```

在 Cache-Aside 模式里，缓存未命中是正常分支。

正确处理方式通常是：查数据库。

---

## 二、错误分类

Go 服务中常见 Redis 相关错误可以分为几类：

| 类型 | 示例 | 含义 |
| --- | --- | --- |
| 缓存未命中 | `redis.Nil` | key 不存在 |
| context 超时 | `context deadline exceeded` | 操作超过本次 context 超时 |
| context 取消 | `context canceled` | 请求被取消 |
| Redis 系统错误 | connection refused、timeout | Redis 或网络异常 |
| JSON 错误 | unmarshal failed | 缓存内容格式异常 |
| 业务错误 | article not found | 数据库也查不到 |

不同错误要走不同分支。

不要把它们都当成一个 `err != nil` 直接返回。

---

## 三、最常见错误写法

错误示例：

```go
val, err := rdb.Get(ctx, key).Result()
if err != nil {
    return nil, err
}
```

问题：如果 key 不存在，`err` 是 `redis.Nil`，这段代码会直接返回错误。

但在缓存读流程中，key 不存在应该去查数据库。

正确写法：

```go
val, err := rdb.Get(ctx, key).Result()
if errors.Is(err, redis.Nil) {
    // cache miss, query database
}
if err != nil {
    // real redis error
}
```

---

## 四、推荐读取缓存分支

```go
val, err := rdb.Get(ctx, key).Result()
if err == nil {
    // cache hit
    return decode(val)
}

if errors.Is(err, redis.Nil) {
    // cache miss
    return loadFromDB(ctx, id)
}

// redis system error
logger.Printf("redis get failed key=%s err=%v", key, err)
return loadFromDB(ctx, id)
```

这个流程体现了：

- 命中：返回缓存。
- 未命中：查数据库。
- Redis 错误：记录日志，降级查数据库。

前提是 Redis 只是缓存层。

如果 Redis 保存的是必须数据，不能简单降级。

---

## 五、使用 errors.Is

推荐使用：

```go
errors.Is(err, redis.Nil)
```

而不是：

```go
err == redis.Nil
```

`errors.Is` 可以处理错误包装。

比如你写了：

```go
return fmt.Errorf("get article cache: %w", redis.Nil)
```

上层仍然可以用：

```go
errors.Is(err, redis.Nil)
```

判断出来。

这符合 Go 的错误包装习惯。

---

## 六、JSON 反序列化错误

读取缓存成功后，还要反序列化：

```go
var article ArticleCacheDTO
if err := json.Unmarshal([]byte(val), &article); err != nil {
    return nil, fmt.Errorf("unmarshal article cache: %w", err)
}
```

这类错误不是缓存未命中，也不是 Redis 故障。

它表示缓存内容坏了或格式不兼容。

常见处理：

```go
logger.Printf("bad cache key=%s err=%v", key, err)
_ = rdb.Del(ctx, key).Err()
return loadFromDB(ctx, id)
```

这样可以删除坏缓存，并从数据库重建。

---

## 七、context 超时错误

如果 Redis 操作超时，可能出现：

```go
context deadline exceeded
```

判断：

```go
if errors.Is(err, context.DeadlineExceeded) {
    logger.Printf("redis timeout key=%s", key)
    return loadFromDB(ctx, id)
}
```

如果请求本身已经取消：

```go
if errors.Is(err, context.Canceled) {
    return nil, err
}
```

要区分：

- Redis 缓存读取超时，可以降级。
- 整个 HTTP 请求已经取消，继续查数据库可能没意义。

实际代码中，是否继续要看调用链设计。

---

## 八、缓存写入失败怎么办

Cache-Aside 读流程中，数据库查到后会回写 Redis。

```go
err := rdb.Set(ctx, key, value, ttl).Err()
```

如果写缓存失败，通常不应该影响本次请求返回数据库结果。

示例：

```go
article := repo.FindByID(ctx, id)

if err := cache.Set(ctx, article); err != nil {
    logger.Printf("set article cache failed id=%d err=%v", id, err)
}

return article, nil
```

原因：数据库已经查到了权威数据，缓存写失败只是性能优化失败。

但要记录日志，否则 Redis 长期写失败你可能不知道。

---

## 九、删除缓存失败怎么办

更新数据库后删除缓存：

```go
err := rdb.Del(ctx, key).Err()
```

如果删除失败，旧缓存可能继续存在。

处理方式：

- 记录错误日志。
- 重试删除。
- 投递补偿任务。
- 依赖 TTL 兜底。

不要悄悄吞掉：

```go
_ = rdb.Del(ctx, key).Err() // 不推荐，无日志
```

删除缓存失败比写缓存失败更需要关注，因为它可能造成旧数据。

---

## 十、缓存层错误是否应该暴露给用户

大多数情况下，如果 Redis 只是缓存层，Redis 错误不应该直接暴露给用户。

例如文章详情：

```text
Redis 失败 -> 查数据库 -> 返回文章
```

用户不需要知道 Redis 失败。

但系统需要知道：

- 日志。
- 指标。
- 告警。

如果 Redis 参与的是核心能力，比如分布式锁、限流、会话状态，错误处理就要更谨慎。

---

## 十一、定义缓存未命中错误是否有必要

有些团队会把 `redis.Nil` 转成自定义错误：

```go
var ErrCacheMiss = errors.New("cache miss")
```

缓存模块内部：

```go
if errors.Is(err, redis.Nil) {
    return nil, ErrCacheMiss
}
```

上层只关心：

```go
if errors.Is(err, ErrCacheMiss) {
    // query database
}
```

这样可以避免业务层直接依赖 `go-redis`。

小项目可以直接判断 `redis.Nil`。

较大的项目可以封装 `ErrCacheMiss`。

---

## 十二、文章缓存错误处理示例

```go
func (s *ArticleService) GetArticle(ctx context.Context, id int64) (*Article, error) {
    article, err := s.cache.Get(ctx, id)
    if err == nil {
        return article, nil
    }

    if errors.Is(err, redis.Nil) {
        return s.loadFromDBAndSetCache(ctx, id)
    }

    if errors.Is(err, context.Canceled) {
        return nil, err
    }

    s.logger.Printf("get article cache failed id=%d err=%v", id, err)
    return s.loadFromDBAndSetCache(ctx, id)
}
```

这里的策略是：

- 命中：返回。
- 未命中：查数据库。
- 请求取消：直接返回。
- Redis 其他错误：记录日志，降级查数据库。

---

## 十三、常见错误

### 1. 把 redis.Nil 当系统错误

这会导致缓存未命中时接口返回 500。

### 2. 把所有 Redis 错误都吞掉

Redis 挂了很久你也不知道。

### 3. JSON 错误继续返回零值对象

可能返回错误业务数据。

### 4. 删除缓存失败不记录

可能造成旧缓存长期存在。

### 5. 不区分缓存读失败和缓存写失败

读失败可能需要降级；写失败通常不影响本次返回，但要记录。

---

## 十四、本节练习

请完成下面练习：

1. 写一个 `GetArticleCache`，在 key 不存在时返回 `redis.Nil`。
2. 在 Service 层使用 `errors.Is(err, redis.Nil)` 判断未命中。
3. Redis 读取超时时记录日志并查数据库。
4. JSON 反序列化失败时删除坏缓存。
5. 缓存写入失败时记录日志，但仍返回数据库结果。
6. 删除缓存失败时记录日志并设计重试方案。
7. 定义一个 `ErrCacheMiss`，把 `redis.Nil` 转换掉。
8. 思考哪些 Redis 错误可以降级，哪些不能。
9. 思考请求取消后是否还要继续查数据库。
10. 写出你自己的缓存错误处理分支表。

---

## 十五、本节小结

这一节你学习了 Go 接 Redis 时的错误处理。

你需要记住：

- `redis.Nil` 表示缓存未命中，不是真实系统错误。
- 推荐使用 `errors.Is(err, redis.Nil)` 判断。
- Redis 系统错误要记录日志，缓存场景通常可以降级查数据库。
- JSON 反序列化失败说明缓存内容异常，必要时删除坏缓存。
- 缓存写入失败通常不影响本次返回，但不能无日志吞掉。
- 删除缓存失败可能导致旧数据，要考虑重试或补偿。
- 错误处理要结合 Redis 在当前业务中的角色来设计。

下一节我们学习 Pipeline 和 Transaction Pipeline，用来减少网络往返并组织批量 Redis 操作。
