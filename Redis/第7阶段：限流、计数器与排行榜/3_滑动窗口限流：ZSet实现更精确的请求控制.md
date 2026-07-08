# 3. 滑动窗口限流：ZSet 实现更精确的请求控制

固定窗口简单，但有边界突刺。

滑动窗口的思想是：不再按固定时间桶统计，而是统计“当前时间往前一段时间内”的真实请求数量。

学完这一节后，你应该能够：

- 理解滑动窗口和固定窗口的区别。
- 使用 ZSet 保存请求时间戳。
- 用 Lua 实现滑动窗口限流。
- 在 Go 中封装滑动窗口限流器。
- 知道滑动窗口的成本和适用场景。

---

## 一、滑动窗口是什么

假设限制：

```text
同一 IP 最近 60 秒最多登录 20 次。
```

当前时间是：

```text
13:02:30
```

滑动窗口统计的是：

```text
13:01:30 到 13:02:30 之间的请求数
```

下一秒变成：

```text
13:01:31 到 13:02:31
```

窗口一直随当前时间滑动。

---

## 二、为什么用 ZSet

ZSet 有两个值：

```text
member：请求唯一标识
score：请求时间戳
```

例如：

```redis
ZADD rate:login:ip:1.2.3.4 1790000000123 req-abc
```

score 使用毫秒时间戳。

每次请求：

```text
1. 删除窗口外的旧请求
2. 统计窗口内请求数
3. 如果数量未超过限制，加入当前请求
4. 设置 key 过期时间
```

---

## 三、Redis 命令流程

假设窗口是 60 秒，当前时间是 `now_ms`。

窗口起点：

```text
window_start = now_ms - 60000
```

命令：

```redis
ZREMRANGEBYSCORE rate:login:ip:1.2.3.4 0 window_start
ZCARD rate:login:ip:1.2.3.4
ZADD rate:login:ip:1.2.3.4 now_ms request_id
EXPIRE rate:login:ip:1.2.3.4 120
```

这些命令要么用 Pipeline 减少 RTT，要么用 Lua 保证原子。

限流场景更推荐 Lua。

---

## 四、滑动窗口 Lua 脚本

```lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]
local ttl = tonumber(ARGV[5])

redis.call("ZREMRANGEBYSCORE", key, 0, now - window)

local count = redis.call("ZCARD", key)
if count >= limit then
    return {0, count}
end

redis.call("ZADD", key, now, member)
redis.call("EXPIRE", key, ttl)

return {1, count + 1}
```

返回：

```text
{1, count}：放行，count 是加入后的窗口请求数
{0, count}：拒绝，count 是当前窗口请求数
```

---

## 五、member 为什么要唯一

ZSet 的 member 是唯一的。

如果使用时间戳当 member：

```text
member = now_ms
```

同一毫秒内多个请求可能覆盖，导致计数偏小。

更安全的 member：

```text
request_id
now_ms + random
trace_id
```

Go 示例：

```go
member := fmt.Sprintf("%d:%s", now.UnixMilli(), uuid.NewString())
```

---

## 六、Go 封装滑动窗口限流器

```go
var slidingWindowScript = redis.NewScript(`
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]
local ttl = tonumber(ARGV[5])

redis.call("ZREMRANGEBYSCORE", key, 0, now - window)

local count = redis.call("ZCARD", key)
if count >= limit then
    return {0, count}
end

redis.call("ZADD", key, now, member)
redis.call("EXPIRE", key, ttl)

return {1, count + 1}
`)
```

```go
type SlidingWindowLimiter struct {
    rdb *redis.Client
}

func NewSlidingWindowLimiter(rdb *redis.Client) *SlidingWindowLimiter {
    return &SlidingWindowLimiter{rdb: rdb}
}
```

```go
func (l *SlidingWindowLimiter) Allow(ctx context.Context, key string, limit int64, window time.Duration) (bool, int64, error) {
    now := time.Now()
    member := fmt.Sprintf("%d:%s", now.UnixMilli(), uuid.NewString())
    ttl := int64((window * 2).Seconds())

    result, err := slidingWindowScript.Run(
        ctx,
        l.rdb,
        []string{key},
        now.UnixMilli(),
        window.Milliseconds(),
        limit,
        member,
        ttl,
    ).Result()
    if err != nil {
        return false, 0, err
    }

    values := result.([]interface{})
    allowed := values[0].(int64) == 1
    count := values[1].(int64)

    return allowed, count, nil
}
```

实际项目里要对类型断言做更严谨的处理。

---

## 七、登录限流示例

```go
key := fmt.Sprintf("rate:login:ip:%s", clientIP)

allow, count, err := limiter.Allow(ctx, key, 20, time.Minute)
if err != nil {
    return err
}
if !allow {
    return ErrTooManyRequests
}

logger.Printf("login rate count ip=%s count=%d", clientIP, count)
```

和固定窗口不同，滑动窗口 key 不需要带具体分钟桶。

因为 ZSet 内部保存了每次请求的时间戳。

---

## 八、滑动窗口的优点

优点：

- 比固定窗口更精确。
- 能缓解窗口边界突刺。
- 可以返回当前窗口真实请求数量。
- 适合登录、短信、敏感操作等风控场景。

例如每分钟 20 次，滑动窗口不会允许用户在边界 1 秒内轻易通过 40 次。

---

## 九、滑动窗口的成本

滑动窗口需要保存每次请求记录。

成本比固定窗口高：

- 每次请求要写 ZSet。
- 每次请求要清理旧数据。
- 热点 key 的 ZSet 可能变大。
- member 唯一值会占用内存。

如果某接口 QPS 很高，滑动窗口要谨慎使用。

可以考虑：

- 固定窗口。
- 令牌桶。
- 本地限流 + Redis 全局限流。
- 网关层限流。

---

## 十、清理窗口外数据

脚本里有：

```redis
ZREMRANGEBYSCORE key 0 now-window
```

这一步很重要。

如果不清理，ZSet 会无限增长。

同时还要给 key 设置 TTL：

```redis
EXPIRE key ttl
```

即使没有请求继续触发清理，key 最终也会消失。

---

## 十一、是否要把被拒绝请求加入窗口

上面的脚本没有把被拒绝请求加入窗口。

含义是：

```text
超过限制后，拒绝请求不继续增加惩罚时间。
```

也可以选择加入被拒绝请求，让恶意请求越刷越久不能通过。

但这可能导致正常用户误操作后等待时间变长。

选择取决于业务：

- 登录爆破：可以更严格。
- 普通按钮防抖：不必过度惩罚。

---

## 十二、常见错误

### 1. member 不唯一

同一毫秒请求互相覆盖，导致计数偏小。

### 2. 不清理旧数据

ZSet 无限增长。

### 3. 没有设置 key TTL

冷门用户的限流 key 长期残留。

### 4. 对高 QPS 全局接口使用精确滑动窗口

内存和 CPU 成本可能过高。

### 5. Redis 错误时无策略

风控接口必须考虑 Redis 故障行为。

---

## 十三、本节练习

请完成下面练习：

1. 解释滑动窗口和固定窗口的区别。
2. 设计一个登录 IP 滑动窗口 key。
3. 写出 ZSet 保存请求时间戳的命令。
4. 写出滑动窗口 Lua 脚本。
5. 解释 member 为什么要唯一。
6. 在 Go 中封装 `Allow` 方法。
7. 思考被拒绝请求是否应该加入窗口。
8. 评估滑动窗口在高 QPS 接口上的成本。

---

## 十四、本节小结

这一节你学习了滑动窗口限流。

你需要记住：

- 滑动窗口统计最近一段时间内的真实请求数。
- ZSet 的 score 保存请求时间戳，member 保存请求唯一标识。
- Lua 可以保证清理、计数、写入和设置 TTL 的原子性。
- 滑动窗口更精确，但比固定窗口成本更高。
- 是否把拒绝请求加入窗口，要根据风控强度决定。

下一节我们会学习令牌桶和漏桶思想，用来控制更平滑的请求速率。

