# 6. 创建接口限流：用户维度与 Lua 优化

创建短链接是写接口。

如果没有限制，恶意用户可以大量创建垃圾短链接，占用数据库、Redis 和短码空间。Redis 非常适合做轻量限流。

学完这一节后，你应该能够：

- 使用 Redis 给 `POST /links` 做用户维度限流。
- 理解 `INCR + EXPIRE` 的固定窗口限流。
- 处理首次访问和 TTL。
- 使用 Lua 优化限流原子性。
- 设计限流失败时的 HTTP 响应。

---

## 一、限流目标

接口：

```text
POST /links
```

规则示例：

```text
每个用户每分钟最多创建 20 个短链接
```

Redis key：

```text
shortlink:create_limit:{user_id}
```

示例：

```text
shortlink:create_limit:1001
```

---

## 二、固定窗口限流

固定窗口的思路：

```text
用户第一次请求 -> INCR 变成 1 -> 设置 60 秒过期
后续请求 -> INCR 递增
超过阈值 -> 拒绝
```

命令：

```redis
INCR shortlink:create_limit:1001
EXPIRE shortlink:create_limit:1001 60
```

优点：

- 简单。
- 性能好。
- 适合学习项目和很多普通业务。

缺点：

- 窗口边界可能突刺。
- `INCR` 和 `EXPIRE` 分开执行时存在极小异常窗口。

---

## 三、基础实现

示例：

```go
func (c *ShortLinkCache) HitCreateLimit(ctx context.Context, userID int64, limit int64, window time.Duration) (bool, error) {
    key := createLimitKey(userID)

    n, err := c.rdb.Incr(ctx, key).Result()
    if err != nil {
        return false, err
    }

    if n == 1 {
        if err := c.rdb.Expire(ctx, key, window).Err(); err != nil {
            return false, err
        }
    }

    return n > limit, nil
}
```

返回值含义：

```text
true  -> 已超过限制
false -> 允许请求
```

---

## 四、创建接口使用限流

服务层流程：

```go
func (s *ShortLinkService) Create(ctx context.Context, userID int64, req CreateLinkRequest) (*CreateLinkResponse, error) {
    limited, err := s.cache.HitCreateLimit(ctx, userID, 20, time.Minute)
    if err != nil {
        log.Printf("create limit failed: user_id=%d err=%v", userID, err)
        return nil, ErrTooManyRequests
    }
    if limited {
        return nil, ErrTooManyRequests
    }

    code := GenerateCode(6)
    link, err := s.repo.Create(ctx, userID, code, req.OriginalURL, req.Title, req.ExpiresAt)
    if err != nil {
        return nil, err
    }

    _ = s.cache.DeleteNotFound(ctx, code)
    return toCreateResponse(link), nil
}
```

限流 Redis 失败时怎么处理，要看业务。

对于创建接口，建议保守拒绝。

因为创建是写入资源的操作，放行可能导致垃圾数据暴涨。

---

## 五、HTTP 响应

超过限制时返回：

```text
HTTP/1.1 429 Too Many Requests
```

响应体：

```json
{
  "error": "too many create requests"
}
```

可以加响应头：

```text
Retry-After: 60
```

告诉客户端稍后再试。

---

## 六、Lua 原子限流

基础实现中，`INCR` 和 `EXPIRE` 是两个命令。

可以用 Lua 合并成原子操作：

```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call("INCR", key)
if current == 1 then
    redis.call("EXPIRE", key, window)
end

if current > limit then
    return 1
end

return 0
```

返回：

```text
1 表示限流
0 表示允许
```

---

## 七、go-redis 执行 Lua

示例：

```go
var createLimitScript = redis.NewScript(`
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call("INCR", key)
if current == 1 then
    redis.call("EXPIRE", key, window)
end

if current > limit then
    return 1
end

return 0
`)

func (c *ShortLinkCache) HitCreateLimitLua(ctx context.Context, userID int64, limit int64, windowSeconds int) (bool, error) {
    result, err := createLimitScript.Run(ctx, c.rdb, []string{createLimitKey(userID)}, limit, windowSeconds).Int()
    if err != nil {
        return false, err
    }
    return result == 1, nil
}
```

Lua 的好处是原子。

但不要把复杂业务都塞进 Lua。

---

## 八、窗口边界问题

固定窗口可能出现边界突刺。

例如：

```text
12:00:59 创建 20 次
12:01:00 又创建 20 次
```

短时间内用户实际创建了 40 次。

如果业务对限流精度要求高，可以考虑：

- 滑动窗口。
- 令牌桶。
- 漏桶。
- Lua + ZSet。

本项目固定窗口已经足够。

---

## 九、常见错误

### 1. 限流 key 不设置过期时间

用户会被永久限制。

### 2. 所有人共用一个限流 key

一个用户会影响所有用户。

### 3. Redis 限流失败时直接放行创建

写接口建议更保守。

### 4. Lua 里写复杂业务逻辑

Lua 应该只做原子 Redis 操作。

### 5. 没有返回 429

客户端无法区分限流和普通错误。

---

## 十、本节练习

请完成下面练习：

1. 实现 `HitCreateLimit`。
2. 在 `POST /links` 开头调用限流。
3. 超过限制时返回 429。
4. 使用 Redis 查看限流 key 的 TTL。
5. 把限流逻辑改成 Lua 版本。
6. 思考 Redis 故障时创建接口应该放行还是拒绝。

---

## 十一、本节小结

这一节完成了创建接口限流。

你需要记住：

- 创建短链接是写接口，必须防滥用。
- 固定窗口限流可以用 `INCR + EXPIRE` 实现。
- 限流 key 必须按用户维度设计。
- Lua 可以保证限流计数和过期设置的原子性。
- 写接口在 Redis 限流失败时通常要保守。

下一节我们处理更新短链接：删除缓存与一致性。

