# 3. 锁 value 唯一标识与误删问题

上一节我们用 `SET key value NX EX seconds` 获取了基础锁。

这一节重点看锁的 value。很多分布式锁事故，不是因为没有加锁，而是释放锁时没有确认锁还属于自己。

学完这一节后，你应该能够：

- 理解锁 value 为什么必须唯一。
- 解释直接 `DEL lock_key` 的风险。
- 设计合理的 request id。
- 区分锁过期、锁释放和锁归属。
- 为下一节 Lua 安全释放锁打基础。

---

## 一、错误释放锁方式

很多初学写法是：

```go
locked, err := rdb.SetNX(ctx, key, "locked", 10*time.Second).Result()
if err != nil {
    return err
}
if !locked {
    return ErrBusy
}

defer rdb.Del(context.Background(), key)

// 执行业务
```

问题有两个：

```text
1. value 是固定值，不能标识持有者。
2. defer 里直接 DEL，可能删除别人的锁。
```

低并发和业务执行很快时，这段代码可能长期看不出问题。

一旦业务执行超过锁 TTL，风险就会出现。

---

## 二、误删别人锁的时间线

假设锁 TTL 是 10 秒。

```text
00s 请求 A 获取锁，value=locked，TTL=10s
02s 请求 A 开始执行业务
10s A 的锁过期
11s 请求 B 获取锁，value=locked，TTL=10s
12s 请求 A 业务完成，执行 DEL lock:key
```

结果：

```text
A 删除了 B 刚刚获取的锁
```

B 还以为自己持有锁，但锁已经没了。

接下来请求 C 也可能获取锁，导致 B 和 C 同时执行业务。

---

## 三、value 要能标识持有者

锁 value 应该是每次加锁唯一的标识。

例如：

```text
request_id = uuid
```

加锁：

```redis
SET lock:order:1001 01J9W6J4Y3R9J3Y9R4Q4Z7 NX EX 10
```

释放锁时检查：

```text
只有 Redis 中的 value 仍然等于我的 request_id，才能删除。
```

这样即使锁过期后被别人重新获取，原请求也不会误删。

---

## 四、request id 怎么生成

常见方案：

```text
UUID
ULID
雪花 ID
instance_id + timestamp + random
trace_id + random
```

Go 示例：

```go
requestID := uuid.NewString()
```

或者不用引入 UUID 库时：

```go
requestID := fmt.Sprintf("%s:%d:%d", instanceID, time.Now().UnixNano(), rand.Int63())
```

推荐使用成熟 ID 库，避免自己拼出来的值碰撞。

---

## 五、value 不是业务用户 ID

不要把锁 value 简单写成用户 ID：

```text
value = user_id
```

同一个用户短时间内可能发起多个请求。

这些请求会产生不同的锁持有过程，但 value 相同，释放时仍然不安全。

value 应该标识“这一次加锁行为”，而不是只标识用户。

---

## 六、安全释放的逻辑

安全释放锁的业务语义是：

```text
如果 lock_key 当前 value 等于我的 request_id：
    删除 lock_key
否则：
    什么也不做
```

伪代码：

```text
if redis.GET(lock_key) == request_id:
    redis.DEL(lock_key)
```

但注意，这两步不能分开执行。

如果先 `GET` 再 `DEL`，中间仍然可能发生锁过期和别人重新获取。

所以需要 Lua 把判断和删除合成一个原子操作。

---

## 七、为什么 GET 后 DEL 仍然不安全

时间线：

```text
请求 A：GET lock:key -> A
锁刚好过期
请求 B：SET lock:key B NX EX 10 成功
请求 A：DEL lock:key
```

即使 A 做了 value 校验，只要校验和删除不是原子的，仍然可能误删。

安全释放必须满足：

```text
GET 和 DEL 在 Redis 里一次完成，中间不能插入其他命令。
```

这就是 Lua 的价值。

---

## 八、Go 中保存 request id

获取锁时需要把 request id 留下来，用于释放锁。

可以定义一个结构体：

```go
type Lock struct {
    Key   string
    Value string
    TTL   time.Duration
}
```

获取锁返回 `*Lock`：

```go
func TryLock(ctx context.Context, rdb *redis.Client, key string, ttl time.Duration) (*Lock, bool, error) {
    value := uuid.NewString()

    ok, err := rdb.SetNX(ctx, key, value, ttl).Result()
    if err != nil {
        return nil, false, err
    }
    if !ok {
        return nil, false, nil
    }

    return &Lock{Key: key, Value: value, TTL: ttl}, true, nil
}
```

这样业务层释放锁时不会丢失 value。

---

## 九、释放失败意味着什么

释放锁时可能出现几种情况：

| 结果 | 含义 |
| --- | --- |
| 删除成功 | 锁仍由自己持有，已释放 |
| 返回 0 | 锁不存在，或锁已不属于自己 |
| Redis 错误 | 无法确认释放结果 |

返回 0 不一定是严重错误。

可能是业务执行太久，锁已经自然过期。

但它是一个重要信号：

```text
你的锁 TTL 可能过短，或者业务执行时间不可控。
```

生产系统应该记录日志和指标。

---

## 十、锁 value 和业务幂等的关系

锁 value 只解决：

```text
释放锁时不要误删别人的锁
```

它不解决：

- 请求重复提交。
- 锁过期后旧请求继续写数据库。
- 服务重试导致重复创建订单。
- Redis 故障时业务如何处理。

这些仍然要靠幂等、数据库约束、状态机和补偿。

不要把唯一 request id 当成完整幂等方案。

---

## 十一、常见错误

### 1. value 固定写成 `1` 或 `locked`

释放时无法判断锁是不是自己持有的。

### 2. value 使用用户 ID

同一用户的不同请求可能互相误判。

### 3. 先 `GET` 再 `DEL`

校验和删除不是原子操作，仍然可能误删。

### 4. 释放失败完全忽略

释放失败可能暴露锁 TTL 过短或 Redis 抖动问题。

### 5. 以为唯一 value 能解决锁超时

唯一 value 只能避免误删，不能阻止过期锁持有者继续执行业务。

---

## 十二、本节练习

请完成下面练习：

1. 画出 A 锁过期后误删 B 锁的时间线。
2. 解释为什么锁 value 不能固定为 `locked`。
3. 解释为什么锁 value 不应该只用用户 ID。
4. 在 Go 中生成一个唯一 request id。
5. 设计一个 `Lock` 结构体，保存 key、value、ttl。
6. 思考释放锁返回 0 时，应该记录哪些日志字段。
7. 思考锁 value 能否解决重复下单问题。

---

## 十三、本节小结

这一节你学习了锁 value 的意义。

你需要记住：

- 锁 value 必须唯一，标识一次具体加锁行为。
- 释放锁不能直接 `DEL`。
- 释放锁必须先判断 value 是否等于自己的 request id。
- `GET` 后再 `DEL` 仍然不是原子的。
- 安全释放锁需要 Lua 脚本。
- 唯一 value 只能防止误删，不能替代幂等和锁超时治理。

下一节我们会正式用 Lua 实现安全释放锁。

