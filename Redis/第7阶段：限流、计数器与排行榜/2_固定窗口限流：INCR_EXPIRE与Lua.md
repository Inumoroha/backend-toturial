# 2. 固定窗口限流：INCR、EXPIRE 与 Lua

固定窗口限流是最容易上手的限流方案。

它的核心思想是：把时间切成固定窗口，在每个窗口内记录请求次数，超过阈值就拒绝。

学完这一节后，你应该能够：

- 理解固定窗口限流的工作方式。
- 使用 `INCR` 和 `EXPIRE` 实现基础限流。
- 理解窗口边界突刺问题。
- 用 Lua 保证计数和设置 TTL 的原子性。
- 在 Go 中封装一个固定窗口限流器。

---

## 一、固定窗口是什么

假设限制：

```text
同一个 IP 每分钟最多登录 20 次。
```

可以把时间按分钟分桶：

```text
2026-07-05 13:01
2026-07-05 13:02
2026-07-05 13:03
```

每分钟一个 Redis key：

```text
rate:login:ip:192.168.1.10:202607051301
```

每次请求：

```text
计数 +1
如果计数 > 20，拒绝
否则放行
```

---

## 二、基础命令实现

最直接的 Redis 命令：

```redis
INCR rate:login:ip:192.168.1.10:202607051301
EXPIRE rate:login:ip:192.168.1.10:202607051301 120
```

为什么 TTL 可以比窗口长一点？

因为窗口结束后，这个 key 只用于统计和短暂排查，很快就可以清理。

例如一分钟窗口，可以设置 120 秒过期。

---

## 三、为什么 INCR 后要设置 EXPIRE

如果不设置过期时间，限流 key 会越来越多：

```text
rate:login:ip:a:202607051301
rate:login:ip:a:202607051302
rate:login:ip:a:202607051303
```

这些 key 过了窗口就没有用了。

必须依靠 TTL 自动清理。

---

## 四、INCR 和 EXPIRE 分开的问题

错误写法：

```go
n, err := rdb.Incr(ctx, key).Result()
if err != nil {
    return err
}
if n == 1 {
    _ = rdb.Expire(ctx, key, 2*time.Minute).Err()
}
```

如果 `INCR` 成功后，服务在 `EXPIRE` 前崩溃，key 可能没有 TTL。

这个风险不是特别高，但在高频限流场景里，最好用 Lua 合成原子操作。

---

## 五、固定窗口 Lua 脚本

```lua
local current = redis.call("INCR", KEYS[1])
if current == 1 then
    redis.call("EXPIRE", KEYS[1], ARGV[1])
end
return current
```

参数：

```text
KEYS[1]：限流 key
ARGV[1]：key 过期秒数
```

含义：

```text
计数 +1；
如果是第一次请求，设置过期时间；
返回当前计数。
```

---

## 六、Go 封装固定窗口限流器

```go
var fixedWindowScript = redis.NewScript(`
local current = redis.call("INCR", KEYS[1])
if current == 1 then
    redis.call("EXPIRE", KEYS[1], ARGV[1])
end
return current
`)
```

```go
type FixedWindowLimiter struct {
    rdb *redis.Client
}

func NewFixedWindowLimiter(rdb *redis.Client) *FixedWindowLimiter {
    return &FixedWindowLimiter{rdb: rdb}
}
```

限流方法：

```go
func (l *FixedWindowLimiter) Allow(ctx context.Context, key string, limit int64, ttl time.Duration) (bool, int64, error) {
    seconds := int64(ttl.Seconds())

    current, err := fixedWindowScript.Run(ctx, l.rdb, []string{key}, seconds).Int64()
    if err != nil {
        return false, 0, err
    }

    return current <= limit, current, nil
}
```

返回：

```text
allow：是否放行
current：当前窗口计数
error：Redis 错误
```

---

## 七、登录 IP 限流 key

按分钟限流：

```go
func LoginIPRateKey(ip string, now time.Time) string {
    bucket := now.Format("200601021504")
    return fmt.Sprintf("rate:login:ip:%s:%s", ip, bucket)
}
```

使用：

```go
key := LoginIPRateKey(clientIP, time.Now())

allow, current, err := limiter.Allow(ctx, key, 20, 2*time.Minute)
if err != nil {
    // 根据业务决定放行、拒绝或走降级策略
}
if !allow {
    return ErrTooManyRequests
}
```

这里 `20` 表示每分钟最多 20 次。

TTL 设为 2 分钟，用来覆盖当前窗口并自动清理。

---

## 八、短信手机号限流 key

短信常见限制：

```text
同一手机号每分钟最多 1 条。
同一手机号每小时最多 5 条。
同一手机号每天最多 10 条。
```

可以用多个 key 同时限制：

```text
rate:sms:phone:13800000000:min:202607051301
rate:sms:phone:13800000000:hour:2026070513
rate:sms:phone:13800000000:day:20260705
```

任何一个维度超限，都拒绝发送。

短信涉及成本和风控，不建议只做一个简单分钟限制。

---

## 九、窗口边界突刺问题

固定窗口的缺点是边界突刺。

例如限制每分钟 20 次：

```text
13:01:59 发送 20 次
13:02:00 再发送 20 次
```

从固定窗口看，每分钟都没超过 20。

但在 1 秒内实际通过了 40 次。

这就是窗口边界问题。

如果业务不能接受这种突刺，可以考虑：

- 缩短窗口，例如每 10 秒限制。
- 多维度组合限制。
- 使用滑动窗口。
- 使用令牌桶。

---

## 十、Redis 错误时怎么办

固定窗口限流的 Redis 操作失败时，不要无脑放行。

要结合业务：

| 场景 | 建议 |
| --- | --- |
| 登录接口 | 可提高验证码难度或拒绝高风险请求 |
| 短信发送 | 倾向拒绝，避免成本失控 |
| 普通查询接口 | 可以放行并记录日志 |
| 管理后台敏感操作 | 倾向保守拒绝 |

代码里可以把策略交给调用方。

限流器只返回错误，不在底层替业务做决定。

---

## 十一、返回重试时间

被限流时，最好告诉调用方多久后再试。

可以读取 TTL：

```go
ttl, err := rdb.TTL(ctx, key).Result()
```

HTTP 接口可以返回：

```text
429 Too Many Requests
Retry-After: 32
```

前端可以展示：

```text
操作太频繁，请 32 秒后再试。
```

---

## 十二、固定窗口适合什么场景

适合：

- 简单接口限流。
- 后台管理操作。
- 登录失败尝试限制。
- 短信小时/天级限制。
- 对窗口边界突刺不敏感的业务。

不适合：

- 对瞬时突刺很敏感的接口。
- 高价值强风控场景。
- 需要非常平滑请求速率的下游保护。

固定窗口胜在简单。

复杂业务可以先固定窗口起步，再根据监控升级。

---

## 十三、常见错误

### 1. 忘记设置 TTL

限流 key 会一直堆积。

### 2. `INCR` 和 `EXPIRE` 分开且不处理失败

可能产生无 TTL 的 key。

### 3. 只做 IP 限流

容易误伤共享出口，也容易被代理池绕过。

### 4. 短信只做分钟限制

攻击者可以慢速刷成本，应该增加小时和天级限制。

### 5. Redis 错误时直接放行所有请求

高风险接口可能被绕过。

---

## 十四、本节练习

请完成下面练习：

1. 为登录 IP 设计每分钟限流 key。
2. 写出固定窗口 Lua 脚本。
3. 用 Go 封装 `Allow` 方法。
4. 给短信手机号设计分钟、小时、天三个限流 key。
5. 解释固定窗口的边界突刺问题。
6. 思考短信接口 Redis 故障时应该怎么处理。
7. 给 HTTP 响应增加 `Retry-After`。

---

## 十五、本节小结

这一节你学习了固定窗口限流。

你需要记住：

- 固定窗口用时间桶记录请求次数。
- `INCR + EXPIRE` 是基础实现。
- 更稳的方式是用 Lua 保证计数和设置 TTL 的原子性。
- 固定窗口简单高效，但有边界突刺。
- 限流要设计维度，Redis 错误时要有业务策略。

下一节我们会学习更精确的滑动窗口限流：ZSet。

