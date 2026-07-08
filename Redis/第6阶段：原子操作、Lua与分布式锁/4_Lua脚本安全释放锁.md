# 4. Lua 脚本安全释放锁

前面两节已经说明：释放分布式锁不能直接 `DEL`，也不能把 `GET` 和 `DEL` 分成两条命令。

这一节我们用 Lua 脚本实现安全释放锁。

学完这一节后，你应该能够：

- 写出安全释放锁 Lua 脚本。
- 在 Go 中用 `redis.NewScript` 执行脚本。
- 正确处理释放锁返回值。
- 理解释放锁失败的业务含义。
- 封装一个基础 `RedisLock`。

---

## 一、安全释放锁脚本

Lua 脚本：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

含义：

```text
KEYS[1]：锁 key
ARGV[1]：当前请求的唯一 value

如果 Redis 中的 value 等于我的 value，就删除锁；
否则不删除。
```

这个脚本在 Redis 内部一次执行，中间不会插入其他命令。

---

## 二、为什么 Lua 可以保证安全释放

如果用两条命令：

```redis
GET lock:key
DEL lock:key
```

两条命令之间可能发生：

- 锁过期。
- 其他请求获取锁。
- 主动删除或修改锁。

Lua 脚本执行期间，Redis 不会执行其他客户端命令。

所以：

```text
读取 value
判断 value
删除 key
```

这三个动作对外表现为一个原子操作。

---

## 三、Go 中使用 redis.NewScript

定义脚本：

```go
var releaseLockScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
`)
```

执行脚本：

```go
func ReleaseLock(ctx context.Context, rdb *redis.Client, key, value string) (bool, error) {
    n, err := releaseLockScript.Run(ctx, rdb, []string{key}, value).Int()
    if err != nil {
        return false, err
    }
    return n == 1, nil
}
```

返回值：

```text
true：释放成功
false：锁不存在，或锁已经不属于自己
error：Redis 执行异常
```

---

## 四、封装 RedisLock

可以把加锁和释放封装到一个类型里：

```go
type RedisLock struct {
    rdb *redis.Client
}

type Lock struct {
    Key   string
    Value string
    TTL   time.Duration
}

func NewRedisLock(rdb *redis.Client) *RedisLock {
    return &RedisLock{rdb: rdb}
}
```

加锁：

```go
func (l *RedisLock) TryLock(ctx context.Context, key string, ttl time.Duration) (*Lock, bool, error) {
    value := uuid.NewString()

    ok, err := l.rdb.SetNX(ctx, key, value, ttl).Result()
    if err != nil {
        return nil, false, err
    }
    if !ok {
        return nil, false, nil
    }

    return &Lock{Key: key, Value: value, TTL: ttl}, true, nil
}
```

释放：

```go
func (l *RedisLock) Unlock(ctx context.Context, lock *Lock) (bool, error) {
    n, err := releaseLockScript.Run(ctx, l.rdb, []string{lock.Key}, lock.Value).Int()
    if err != nil {
        return false, err
    }
    return n == 1, nil
}
```

---

## 五、释放锁要不要用 defer

可以用，但要注意记录释放结果。

```go
lock, ok, err := locker.TryLock(ctx, key, 10*time.Second)
if err != nil {
    return err
}
if !ok {
    return ErrBusy
}

defer func() {
    unlockCtx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()

    ok, err := locker.Unlock(unlockCtx, lock)
    if err != nil {
        logger.Printf("unlock failed key=%s err=%v", lock.Key, err)
        return
    }
    if !ok {
        logger.Printf("unlock skipped key=%s reason=not_owner_or_expired", lock.Key)
    }
}()
```

这里释放锁使用了新的短超时 context。

原因是原请求的 `ctx` 可能已经取消。

释放锁是清理动作，通常仍然应该尝试执行。

---

## 六、业务代码示例

```go
func (s *OrderService) HandleOrder(ctx context.Context, orderID int64) error {
    key := fmt.Sprintf("lock:order:%d", orderID)

    lockCtx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
    defer cancel()

    lock, ok, err := s.locker.TryLock(lockCtx, key, 10*time.Second)
    if err != nil {
        return err
    }
    if !ok {
        return ErrOrderProcessing
    }

    defer func() {
        unlockCtx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
        defer cancel()
        _, _ = s.locker.Unlock(unlockCtx, lock)
    }()

    return s.processOrder(ctx, orderID)
}
```

这只是基础示例。

真实项目还要考虑锁超时、幂等和数据库状态约束。

---

## 七、释放失败如何处理

释放失败分两种。

第一种是 Redis 错误：

```text
网络超时、连接失败、Redis 抖动
```

这时无法确认锁是否释放成功。

不过锁有 TTL，最终会自动过期。

第二种是脚本返回 0：

```text
锁不存在，或 value 不匹配
```

可能原因：

- 业务执行时间超过 TTL。
- 锁已经自动过期。
- 锁被其他流程删除。
- 代码重复释放。

脚本返回 0 应该记录日志和指标。

它提示你检查锁 TTL 和业务耗时。

---

## 八、是否应该 panic

释放锁失败通常不应该 panic。

因为 panic 可能影响正常业务返回，甚至触发上层重试，造成更复杂的问题。

更合理的处理是：

- 记录日志。
- 上报指标。
- 让锁 TTL 兜底过期。
- 排查为什么释放失败。

如果这是强一致业务，仅靠 Redis 锁本身也不够，需要数据库层防线。

---

## 九、Redis Cluster 中的注意点

释放锁脚本只访问一个 key：

```text
KEYS[1]
```

在 Redis Cluster 中没有跨槽问题。

如果脚本访问多个 key，需要保证这些 key 在同一个 hash slot。

可以使用 hash tag：

```text
lock:{order:1001}
state:{order:1001}
```

同一个 `{order:1001}` 内的内容会被用来计算槽位。

---

## 十、常见错误

### 1. 释放锁直接 `DEL`

可能误删别人持有的锁。

### 2. 先 `GET` 再 `DEL`

判断和删除之间不是原子操作。

### 3. 释放锁使用已取消的请求 context

请求超时后，释放锁可能直接不执行。

### 4. 忽略脚本返回 0

这可能表示锁已经过期或不属于自己。

### 5. Lua 脚本里写复杂业务

释放锁脚本应该短小明确。

---

## 十一、本节练习

请完成下面练习：

1. 写出安全释放锁 Lua 脚本。
2. 解释 `KEYS[1]` 和 `ARGV[1]` 的含义。
3. 在 Go 中用 `redis.NewScript` 封装脚本。
4. 实现 `Unlock(ctx, lock)`。
5. 让释放锁使用 100ms 短超时。
6. 释放锁返回 0 时打印 key、value、ttl。
7. 思考为什么释放锁失败不应该直接 panic。
8. 思考 Redis Cluster 中多个 key 脚本有什么限制。

---

## 十二、本节小结

这一节你完成了基础分布式锁的安全释放。

你需要记住：

- 释放锁必须校验 value。
- 校验和删除必须用 Lua 合成原子操作。
- `redis.NewScript` 是 Go 中执行 Lua 的推荐封装。
- 释放锁要处理 Redis 错误和返回 0 两种情况。
- 释放锁可以使用新的短超时 context。
- 安全释放锁仍然不能解决锁超时和业务幂等问题。

下一节我们会继续讨论最麻烦的问题：业务执行时间超过锁 TTL。

