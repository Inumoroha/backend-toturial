# 4. 短链接详情缓存：Cache-Aside 与空值缓存

这一节实现短链接详情缓存。

短链接跳转是高频读接口。每次都查数据库会增加延迟，也会让数据库承受不必要的压力。Cache-Aside 是最常见、最适合入门的缓存模式。

学完这一节后，你应该能够：

- 理解 Cache-Aside 的读流程。
- 使用 Redis 缓存短链接详情。
- 使用 JSON 序列化缓存对象。
- 使用空值缓存防止缓存穿透。
- 为详情缓存和空值缓存设置 TTL。

---

## 一、Cache-Aside 是什么

Cache-Aside 也叫旁路缓存。

读流程：

```text
先读缓存
命中 -> 返回
未命中 -> 查数据库
数据库存在 -> 写缓存 -> 返回
数据库不存在 -> 写空值缓存 -> 返回不存在
```

写流程：

```text
先写数据库
再删除缓存
```

在短链接项目中，跳转接口和详情接口都可以复用这套读取逻辑。

---

## 二、缓存对象

缓存对象不需要包含数据库表全部字段。

```go
type CachedShortLink struct {
    Code        string     `json:"code"`
    OriginalURL string     `json:"original_url"`
    Status      string     `json:"status"`
    ExpiresAt   *time.Time `json:"expires_at,omitempty"`
}
```

判断短链接是否可用时，需要：

- `OriginalURL`
- `Status`
- `ExpiresAt`

其他字段可以查询详情接口再补。

---

## 三、读取详情缓存

示例：

```go
func (c *ShortLinkCache) GetDetail(ctx context.Context, code string) (*CachedShortLink, error) {
    val, err := c.rdb.Get(ctx, detailKey(code)).Result()
    if errors.Is(err, redis.Nil) {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }

    var link CachedShortLink
    if err := json.Unmarshal([]byte(val), &link); err != nil {
        return nil, err
    }

    return &link, nil
}
```

注意：

- `redis.Nil` 表示缓存不存在。
- JSON 反序列化失败要记录日志。
- 不要把缓存不存在当成接口失败。

---

## 四、写入详情缓存

示例：

```go
func (c *ShortLinkCache) SetDetail(ctx context.Context, link CachedShortLink, ttl time.Duration) error {
    b, err := json.Marshal(link)
    if err != nil {
        return err
    }
    return c.rdb.Set(ctx, detailKey(link.Code), b, ttl).Err()
}
```

TTL 示例：

```go
const detailTTL = 30 * time.Minute
```

详情缓存不建议永久保存。

设置 TTL 可以避免缓存长期持有旧数据，也方便异常情况下自动恢复。

---

## 五、空值缓存

缓存穿透是指请求一直访问不存在的数据。

例如攻击者不断请求：

```text
/abc001
/abc002
/abc003
```

如果这些短码都不存在，每次缓存都未命中，每次都会查数据库。

解决方式之一是空值缓存。

当数据库也查不到时，写入一个短 TTL 标记：

```text
shortlink:not_found:{code}
```

下一次再访问同一个不存在短码时，直接返回 404。

---

## 六、空值缓存方法

示例：

```go
func (c *ShortLinkCache) IsNotFound(ctx context.Context, code string) (bool, error) {
    n, err := c.rdb.Exists(ctx, notFoundKey(code)).Result()
    if err != nil {
        return false, err
    }
    return n > 0, nil
}

func (c *ShortLinkCache) SetNotFound(ctx context.Context, code string, ttl time.Duration) error {
    return c.rdb.Set(ctx, notFoundKey(code), "1", ttl).Err()
}
```

TTL 示例：

```go
const notFoundTTL = 60 * time.Second
```

空值缓存 TTL 要短。

否则用户刚创建某个短码，旧的空值缓存可能导致短时间内仍然返回 404。

---

## 七、读取服务逻辑

服务层伪代码：

```go
func (s *ShortLinkService) GetForRedirect(ctx context.Context, code string) (*CachedShortLink, error) {
    link, err := s.cache.GetDetail(ctx, code)
    if err == nil && link != nil {
        return link, nil
    }
    if err != nil {
        log.Printf("get detail cache failed: code=%s err=%v", code, err)
    }

    hitNotFound, err := s.cache.IsNotFound(ctx, code)
    if err == nil && hitNotFound {
        return nil, ErrNotFound
    }
    if err != nil {
        log.Printf("get not found cache failed: code=%s err=%v", code, err)
    }

    dbLink, err := s.repo.FindByCode(ctx, code)
    if errors.Is(err, ErrNotFound) {
        _ = s.cache.SetNotFound(ctx, code, notFoundTTL)
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, err
    }

    cached := toCachedShortLink(dbLink)
    _ = s.cache.SetDetail(ctx, cached, detailTTL)
    return &cached, nil
}
```

这个逻辑里，Redis 读失败不直接中断主流程，而是回源数据库。

---

## 八、过期和禁用判断

拿到缓存后也要判断状态。

```go
func IsAvailable(link *CachedShortLink, now time.Time) bool {
    if link.Status != "active" {
        return false
    }
    if link.ExpiresAt != nil && now.After(*link.ExpiresAt) {
        return false
    }
    return true
}
```

如果短链接已经禁用或过期，跳转接口不应该继续跳转。

---

## 九、缓存 TTL 抖动

如果大量 key 同时过期，会造成数据库回源峰值。

可以给 TTL 加一点随机抖动：

```go
func detailTTLWithJitter() time.Duration {
    return 30*time.Minute + time.Duration(rand.Intn(300))*time.Second
}
```

这样缓存不会在同一秒集中失效。

---

## 十、常见错误

### 1. 数据库查不到时什么都不缓存

不存在的短码会持续打到数据库。

### 2. 空值缓存 TTL 太长

新创建短码可能短时间不可见。

### 3. Redis 失败直接返回 500

详情缓存失败时通常可以查数据库降级。

### 4. 缓存反序列化失败不处理

应该记录日志，并考虑删除坏缓存。

### 5. 缓存对象缺少状态字段

禁用链接可能继续跳转。

---

## 十一、本节练习

请完成下面练习：

1. 定义 `CachedShortLink`。
2. 实现 `GetDetail` 和 `SetDetail`。
3. 实现 `IsNotFound` 和 `SetNotFound`。
4. 在服务层实现 Cache-Aside 读取流程。
5. 给详情缓存 TTL 加随机抖动。
6. 写出 Redis 故障时的降级策略。

---

## 十二、本节小结

这一节实现了短链接详情缓存。

你需要记住：

- Cache-Aside 是先读缓存，未命中再查数据库。
- 数据库查不到时要写空值缓存。
- 空值缓存 TTL 要短。
- Redis 失败时可以回源数据库降级。
- 缓存对象要包含跳转判断所需字段。

下一节我们实现跳转接口：缓存读取、访问计数与降级。

