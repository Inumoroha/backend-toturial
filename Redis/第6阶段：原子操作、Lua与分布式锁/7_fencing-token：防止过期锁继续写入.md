# 7. fencing token：防止过期锁继续写入

前面我们已经知道：锁过期后，原持有者可能还在继续执行业务。

安全释放锁和 Watch Dog 都不能彻底解决这个问题。fencing token 是一种常见的防线：每次获取锁时发放一个递增令牌，下游写入时只接受更大的令牌。

学完这一节后，你应该能够：

- 理解 fencing token 解决什么问题。
- 用 Redis `INCR` 生成递增 token。
- 在数据库写入时校验 token。
- 区分锁 value 和 fencing token。
- 判断 fencing token 的适用场景。

---

## 一、锁过期后的旧请求问题

时间线：

```text
00s 请求 A 获取锁
01s 请求 A 读取数据，准备写入
10s A 的锁过期
11s 请求 B 获取锁
12s 请求 B 写入数据库
13s 请求 A 恢复执行，也写入数据库
```

如果没有额外防线，A 的旧写入可能覆盖 B 的新写入。

这个问题的本质是：

```text
锁失效后，旧持有者仍然有能力写下游资源。
```

---

## 二、什么是 fencing token

fencing token 是一个单调递增的数字。

每次成功获取锁时，发放一个更大的 token：

```text
A 获取锁 -> token=101
B 获取锁 -> token=102
C 获取锁 -> token=103
```

下游资源只接受 token 更大的写入。

如果 A 过期后拿着 `101` 再来写，而 B 已经用 `102` 写过，下游会拒绝 A。

---

## 三、token 和锁 value 的区别

锁 value：

```text
随机唯一，用来证明“这把锁是我持有的”
```

fencing token：

```text
递增有序，用来证明“我的写入比旧持有者更新”
```

二者用途不同。

不要用 UUID 当 fencing token，因为 UUID 没有顺序。

也不要用 fencing token 替代锁 value，因为释放锁仍然需要唯一 value 校验。

---

## 四、如何用 Redis 生成 token

可以使用 `INCR`：

```redis
INCR lock:token:order:1001
```

Redis 单条 `INCR` 是原子的，会返回递增数字。

获取锁成功后生成 token：

```text
SET lock:order:1001 request_id NX EX 10
INCR lock:token:order:1001
```

但注意：如果 `SET` 成功后 `INCR` 失败，流程会变复杂。

更严谨的方式是用 Lua 把“加锁 + 发 token”放在一起。

---

## 五、加锁并发放 token 的 Lua

```lua
if redis.call("SET", KEYS[1], ARGV[1], "NX", "PX", ARGV[2]) then
    return redis.call("INCR", KEYS[2])
else
    return 0
end
```

含义：

```text
KEYS[1]：锁 key
KEYS[2]：token key
ARGV[1]：锁 value
ARGV[2]：锁 TTL，毫秒

加锁成功后返回递增 token；
加锁失败返回 0。
```

在 Redis Cluster 中，两个 key 要在同一个 hash slot。

可以这样设计：

```text
lock:{order:1001}
lock_token:{order:1001}
```

---

## 六、Go 封装示例

```go
var lockWithTokenScript = redis.NewScript(`
if redis.call("SET", KEYS[1], ARGV[1], "NX", "PX", ARGV[2]) then
    return redis.call("INCR", KEYS[2])
else
    return 0
end
`)
```

```go
type LockWithToken struct {
    Key   string
    Value string
    TTL   time.Duration
    Token int64
}

func (l *RedisLock) TryLockWithToken(ctx context.Context, lockKey, tokenKey string, ttl time.Duration) (*LockWithToken, bool, error) {
    value := uuid.NewString()
    ttlMillis := strconv.FormatInt(ttl.Milliseconds(), 10)

    token, err := lockWithTokenScript.Run(ctx, l.rdb, []string{lockKey, tokenKey}, value, ttlMillis).Int64()
    if err != nil {
        return nil, false, err
    }
    if token == 0 {
        return nil, false, nil
    }

    return &LockWithToken{
        Key:   lockKey,
        Value: value,
        TTL:   ttl,
        Token: token,
    }, true, nil
}
```

---

## 七、数据库如何校验 token

假设订单处理记录表有字段：

```sql
fencing_token BIGINT NOT NULL DEFAULT 0
```

写入时带条件：

```sql
UPDATE order_process
SET status = ?, fencing_token = ?
WHERE order_id = ?
  AND fencing_token < ?;
```

参数中两个 token 都是本次 token。

含义：

```text
只有本次 token 大于数据库里已记录的 token，才允许写入。
```

如果影响 0 行，说明有更新的持有者已经写过。

旧请求应该停止后续动作。

---

## 八、外部资源也要支持校验

fencing token 的关键在于：

```text
被写入的下游资源必须能识别并拒绝旧 token。
```

数据库可以通过条件更新实现。

但如果下游是：

- 第三方 HTTP API。
- 文件系统。
- 不支持条件写入的服务。
- 没有版本字段的旧系统。

fencing token 就很难发挥作用。

这时要考虑别的方案，比如把写入收敛到数据库状态机，或通过单消费者队列串行处理。

---

## 九、fencing token 的适用场景

适合：

- 锁保护的是数据库某行或某个资源。
- 下游写入支持版本号或条件更新。
- 需要防止旧持有者覆盖新持有者。
- 业务对并发顺序敏感。

不适合：

- 下游无法校验 token。
- 只是轻量缓存重建。
- 业务本身已经有可靠状态机。
- 使用消息队列单消费者已经串行化。

---

## 十、token key 会不会无限增长

`INCR` 的 token 会一直增长。

一般来说这不是问题，`int64` 空间足够大。

但 token key 的生命周期要设计清楚：

- 长期资源可以长期保留 token key。
- 短期资源可以在业务结束后设置较长 TTL。
- 不要因为删除 token key 导致 token 从 1 重新开始，从而让旧 token 混淆。

如果资源生命周期结束，可以清理。

如果资源可能复用同一个 ID，要非常谨慎。

---

## 十一、常见错误

### 1. 用 UUID 当 fencing token

UUID 唯一但没有递增顺序。

### 2. 只生成 token，但下游不校验

下游不拒绝旧 token，token 就只是日志字段。

### 3. token key 随便删除

删除后 token 从小值重新开始，可能破坏顺序语义。

### 4. 加锁和发 token 分两步但不处理失败

可能加锁成功却没有 token。

### 5. 以为 fencing token 替代幂等

它防旧写覆盖，不等于处理重复请求。

---

## 十二、本节练习

请完成下面练习：

1. 画出旧请求锁过期后覆盖新请求写入的时间线。
2. 解释锁 value 和 fencing token 的区别。
3. 用 Redis 命令生成递增 token。
4. 写出加锁并发放 token 的 Lua 脚本。
5. 设计 Redis Cluster 下同槽 key。
6. 为一张表增加 `fencing_token` 字段。
7. 写出带 token 条件的 `UPDATE`。
8. 思考第三方 API 不支持 token 时怎么办。

---

## 十三、本节小结

这一节你学习了 fencing token。

你需要记住：

- 锁过期后，旧持有者可能继续写入。
- fencing token 是每次获取锁时发放的递增令牌。
- 下游只接受更大的 token，从而拒绝旧持有者写入。
- 锁 value 用于安全释放锁，fencing token 用于保护下游写入顺序。
- fencing token 必须依赖下游资源的条件写入能力。

下一节我们会了解 Redlock 的基本思想和争议。

