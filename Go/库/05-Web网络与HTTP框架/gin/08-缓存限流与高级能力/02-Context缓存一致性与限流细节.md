# 08-02 Context、缓存一致性与限流细节

本节目标：把 Redis 和 context 讲得更贴近实战。缓存不是简单 `Get` 和 `Set`，还要考虑一致性、过期时间、失败降级和请求取消。

## 一、context 为什么重要

一次 HTTP 请求有生命周期。客户端断开、请求超时、服务关闭时，后端应该尽快停止无意义工作。

Gin 中获取：

```go
ctx := c.Request.Context()
```

传给 service：

```go
user, err := h.userService.GetUser(ctx, userID)
```

传给 Redis：

```go
s.redis.Get(ctx, key).Result()
```

传给 GORM：

```go
s.db.WithContext(ctx).First(&user, id)
```

这样请求取消时，数据库和 Redis 操作也有机会被取消。

## 二、缓存失败怎么办

Redis 只是优化，不应该让核心业务完全依赖缓存。

查询用户详情时：

- Redis 命中，直接返回。
- Redis 挂了，记录日志，继续查数据库。
- 数据库查到后，尝试写缓存。
- 写缓存失败，不影响本次接口成功。

不要因为 Redis 短暂不可用，就让所有用户详情接口失败。

## 三、缓存一致性策略

常见策略：

```text
读：先缓存，未命中查数据库，再写缓存
写：先更新数据库，再删除缓存
```

为什么不是更新数据库后直接更新缓存？

因为更新缓存可能遇到并发覆盖问题。学习阶段优先使用删除缓存，下一次读取再重建。

更新用户：

```go
if err := s.repo.Update(ctx, user); err != nil {
    return err
}

_ = s.redis.Del(ctx, fmt.Sprintf("user:detail:%d", user.ID)).Err()
```

删除缓存失败怎么办？学习阶段可以记录日志，生产环境可能需要重试或消息队列兜底。

## 四、缓存过期时间

不要所有 key 都设置同样过期时间。可以加随机抖动，降低同时过期风险：

```go
ttl := 10*time.Minute + time.Duration(rand.Intn(60))*time.Second
s.redis.Set(ctx, key, value, ttl)
```

热点数据可以更长，变化频繁的数据应该更短。

## 五、限流粒度

限流可以按不同维度：

- 按 IP 限制。
- 按用户 ID 限制。
- 按邮箱限制登录失败。
- 按接口路径限制。

登录失败适合按邮箱或 IP：

```text
login:fail:email:alice@example.com
login:fail:ip:127.0.0.1
```

用户接口适合按用户 ID：

```text
rate:user:123:/api/v1/tasks
```

## 六、简单固定窗口限流

示例：一分钟最多 60 次。

```go
func RateLimit(redis *redis.Client, limit int, window time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := "rate:" + c.ClientIP() + ":" + c.FullPath()
        ctx := c.Request.Context()

        count, err := redis.Incr(ctx, key).Result()
        if err != nil {
            c.Next()
            return
        }
        if count == 1 {
            redis.Expire(ctx, key, window)
        }
        if count > int64(limit) {
            response.Fail(c, http.StatusTooManyRequests, 42901, "too many requests")
            c.Abort()
            return
        }

        c.Next()
    }
}
```

固定窗口简单易懂，但窗口边界可能有突刺。后续可以学习滑动窗口或令牌桶。

## 七、本节练习

完成：

- service 和 repository 方法增加 `context.Context`。
- GORM 查询使用 `WithContext`。
- Redis 缓存失败时不影响数据库查询。
- 更新用户后删除用户详情缓存。
- 给登录接口增加失败次数限制。
- 给一个测试接口增加 IP 维度限流。

## 八、验收清单

你应该能够回答：

- 为什么 context 要从 handler 一路传到 repository？
- Redis 失败时接口一定要失败吗？
- 为什么更新数据库后常删除缓存？
- TTL 加随机值有什么意义？
- 固定窗口限流有什么缺点？

