# 3. context.Context：超时、取消与 Redis 操作

Go 服务里使用 Redis 时，不要忽略 `context.Context`。

`context` 可以传递请求取消信号和超时控制。HTTP 请求已经取消或超时后，继续等待 Redis 没有意义；如果 Redis 异常，也不能让请求无限等待。

学完这一节后，你应该能够：

- 理解为什么 Redis 操作要传 `context.Context`。
- 使用 `context.WithTimeout` 控制单次 Redis 操作耗时。
- 在 HTTP 请求链路中传递 `r.Context()`。
- 区分 client 级超时和 request context 超时。
- 知道什么时候应该给 Redis 单独设置更短的 timeout。

---

## 一、go-redis 的命令都需要 context

`go-redis/v9` 的命令通常都长这样：

```go
val, err := rdb.Get(ctx, key).Result()
```

第一个参数就是 `context.Context`。

常见命令：

```go
rdb.Set(ctx, key, value, ttl)
rdb.Get(ctx, key)
rdb.Del(ctx, key)
rdb.HGet(ctx, key, field)
rdb.ZRevRange(ctx, key, start, stop)
```

这不是摆设。

它让 Redis 操作能响应超时、取消和请求生命周期。

---

## 二、为什么不能一直用 context.Background

本地 demo 里经常写：

```go
ctx := context.Background()
rdb.Get(ctx, key)
```

这在 `main` 函数或启动检查里可以。

但在 HTTP 请求处理中，不应该无脑用 `context.Background()`。

错误示例：

```go
func (h *Handler) GetArticle(w http.ResponseWriter, r *http.Request) {
    ctx := context.Background()
    h.cache.GetArticle(ctx, id)
}
```

这样会丢失 HTTP 请求的取消信号。

如果客户端已经断开连接，后端仍然可能继续等 Redis 和数据库。

正确做法：

```go
func (h *Handler) GetArticle(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    h.cache.GetArticle(ctx, id)
}
```

---

## 三、请求取消是什么意思

用户请求接口：

```text
GET /articles/1001
```

如果用户关闭页面、网络断开、反向代理超时，HTTP 请求的 context 可能被取消。

这时继续执行：

```text
查 Redis
查数据库
序列化响应
```

通常已经没有意义。

`context` 的作用就是把“请求已经结束”的信号传下去。

---

## 四、给 Redis 操作设置超时

有时你希望 Redis 操作比整个 HTTP 请求更短。

比如 HTTP 请求总超时是 1 秒，但 Redis 缓存读取最多只等 50ms。

可以这样写：

```go
func (c *ArticleCache) Get(ctx context.Context, id int64) (*Article, error) {
    redisCtx, cancel := context.WithTimeout(ctx, 50*time.Millisecond)
    defer cancel()

    val, err := c.rdb.Get(redisCtx, articleKey(id)).Result()
    if err != nil {
        return nil, err
    }

    // decode val
    return article, nil
}
```

这样即使上层请求还没超时，Redis 操作也不会无限拖住。

---

## 五、什么时候需要单独 Redis timeout

适合给 Redis 单独设置短 timeout 的场景：

- Redis 是缓存层，不是权威数据源。
- Redis 超时后可以降级查数据库。
- 接口对延迟敏感。
- Redis 偶发抖动不能拖慢整个请求。

比如文章详情：

```text
Redis 50ms 内没返回 -> 记录日志 -> 查数据库
```

但如果 Redis 保存的是必须数据，比如某些限流、锁或会话状态，不能简单忽略错误，要根据业务处理。

---

## 六、client 超时和 context 超时的区别

上一节提到 client 配置：

```go
ReadTimeout:  1 * time.Second,
WriteTimeout: 1 * time.Second,
PoolTimeout:  2 * time.Second,
```

这些是 client 级默认限制。

`context.WithTimeout` 是单次操作或请求级限制。

两者关系：

```text
谁先到期，谁先取消操作
```

比如：

```text
ReadTimeout = 1s
context timeout = 50ms
```

那么这次操作最多等约 50ms。

如果没有 context timeout，client timeout 仍然会保护操作不无限等待。

实际项目里两者都应该合理配置。

---

## 七、不要在函数内部随便覆盖 context

不推荐：

```go
func (c *Cache) GetArticle(ctx context.Context, id int64) {
    ctx = context.Background()
    c.rdb.Get(ctx, key)
}
```

这会丢失上层传来的取消信号。

更好的做法：

```go
func (c *Cache) GetArticle(ctx context.Context, id int64) {
    redisCtx, cancel := context.WithTimeout(ctx, 50*time.Millisecond)
    defer cancel()

    c.rdb.Get(redisCtx, key)
}
```

也就是说：以传入的 ctx 为父 context，再派生更短超时。

---

## 八、context deadline exceeded 怎么处理

如果 Redis 操作因为 context 超时，可能返回：

```text
context deadline exceeded
```

这表示操作超过了 context 设置的 deadline。

处理方式取决于业务。

缓存读取：

```text
记录日志 -> 降级查数据库
```

缓存写入：

```text
记录日志 -> 不影响主流程，或投递补偿任务
```

分布式锁：

```text
获取锁失败 -> 返回稍后重试或业务失败
```

不要把所有 Redis 错误都简单吞掉，也不要所有错误都让接口失败。

---

## 九、HTTP Handler 示例

```go
type Handler struct {
    svc *ArticleService
}

func (h *Handler) GetArticle(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    article, err := h.svc.GetArticle(ctx, 1001)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    _ = json.NewEncoder(w).Encode(article)
}
```

Service 继续传递 context：

```go
func (s *ArticleService) GetArticle(ctx context.Context, id int64) (*Article, error) {
    article, err := s.cache.GetArticle(ctx, id)
    if err == nil {
        return article, nil
    }

    return s.repo.FindByID(ctx, id)
}
```

从 Handler 到 Service，再到 Cache 和 Repository，都应该传递同一个请求 context。

---

## 十、缓存降级示例

缓存读取超时后降级查数据库：

```go
func (s *ArticleService) GetArticle(ctx context.Context, id int64) (*Article, error) {
    article, err := s.cache.GetArticle(ctx, id)
    if err == nil {
        return article, nil
    }

    if errors.Is(err, redis.Nil) {
        return s.loadFromDBAndSetCache(ctx, id)
    }

    // Redis 系统异常：记录日志，允许降级到数据库
    s.logger.Printf("get article cache failed: %v", err)
    return s.repo.FindByID(ctx, id)
}
```

注意：`redis.Nil` 表示缓存未命中，不是 Redis 故障。

错误处理会在后面单独讲。

---

## 十一、常见错误

### 1. HTTP 请求里使用 context.Background

这会丢失请求取消信号。

### 2. Redis 操作没有超时意识

Redis 抖动时可能拖慢整个接口。

### 3. 在函数内部覆盖传入 ctx

会破坏调用链的取消和超时控制。

### 4. 所有 Redis 超时都直接返回 500

如果 Redis 只是缓存层，很多场景可以降级查数据库。

### 5. 把 redis.Nil 和 context 超时混在一起

`redis.Nil` 是未命中；`context deadline exceeded` 是超时。

---

## 十二、本节练习

请完成下面练习：

1. 写一个 HTTP Handler，使用 `r.Context()`。
2. 在 Service 层继续传递 `ctx`。
3. 在 Cache 层使用 `context.WithTimeout(ctx, 50*time.Millisecond)`。
4. Redis `GET` 超时时记录日志。
5. Redis 超时后降级查数据库。
6. 尝试用 `context.Background()` 替代 `r.Context()`，思考问题。
7. 区分 client 的 `ReadTimeout` 和单次 `context.WithTimeout`。
8. 思考文章详情缓存读取最多应该等多久。
9. 思考分布式锁获取超时是否能直接忽略。
10. 写一句你对“请求取消后不应继续等待 Redis”的理解。

---

## 十三、本节小结

这一节你学习了 Go 中使用 Redis 时的 `context`。

你需要记住：

- `go-redis/v9` 命令都应该传入 `context.Context`。
- HTTP 请求中使用 `r.Context()`，不要无脑用 `context.Background()`。
- 可以用 `context.WithTimeout` 给 Redis 操作设置更短超时。
- client 级超时和 context 超时会一起生效。
- Redis 只是缓存层时，超时可以降级查数据库。
- `redis.Nil`、context 超时、真实系统错误要分开处理。

下一节我们会学习常用 Redis 数据结构命令在 Go 中的写法，包括 String、Hash、List、Set、ZSet。
