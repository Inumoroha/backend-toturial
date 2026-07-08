# 5. 锁超时问题与锁续期 Watch Dog

Redis 分布式锁必须设置 TTL，否则持有者崩溃后会形成死锁。

但 TTL 又带来另一个问题：业务还没执行完，锁已经过期了。

这一节讨论锁超时，以及自动续期 Watch Dog 的设计思路。

学完这一节后，你应该能够：

- 理解锁 TTL 过短会发生什么。
- 估算锁 TTL 的基本方法。
- 理解自动续期 Watch Dog 的作用。
- 知道续期必须校验锁 value。
- 判断什么时候不应该依赖续期。

---

## 一、锁过期的危险时间线

假设锁 TTL 是 10 秒。

```text
00s 请求 A 获取锁
01s 请求 A 开始执行业务
10s 锁自动过期
11s 请求 B 获取锁
12s 请求 A 继续写数据库
13s 请求 B 也写数据库
```

结果：

```text
A 和 B 同时进入了临界区
```

唯一 value 和 Lua 释放锁只能避免 A 删除 B 的锁。

它不能阻止 A 在锁过期后继续执行业务。

---

## 二、锁 TTL 不是越长越好

TTL 太短：

- 业务没执行完锁就过期。
- 多个请求可能同时执行业务。
- 释放锁时经常返回 0。

TTL 太长：

- 持有者崩溃后，其他请求要等很久。
- 资源被长时间占用。
- 用户体验变差。

所以 TTL 要结合业务耗时估算。

---

## 三、如何估算锁 TTL

可以从这些数据出发：

- 正常平均耗时。
- P95、P99 耗时。
- 下游数据库或接口抖动时间。
- GC 停顿或容器调度抖动。
- 是否有重试。

示例：

```text
业务 P99 耗时：800ms
偶发下游抖动：1s
预留缓冲：2s

锁 TTL 可以先设为 5s
```

不要拍脑袋设置 `10s` 或 `30s`。

上线后要观察：

- 释放锁返回 0 的次数。
- 业务执行耗时分布。
- 获取锁失败次数。
- 锁等待时间。

---

## 四、什么是 Watch Dog

Watch Dog 是自动续期机制。

基本思路：

```text
1. 请求获取锁，TTL=10s
2. 后台 goroutine 每隔 3s 检查一次
3. 如果锁仍然属于自己，就把 TTL 续回 10s
4. 业务完成后停止续期并释放锁
```

这样可以降低业务执行时间超过 TTL 的风险。

很多成熟 Redis 锁库都有类似机制。

---

## 五、续期也必须校验 value

续期不能简单：

```redis
EXPIRE lock:key 10
```

如果锁已经过期，并且被别人获取，原请求再执行 `EXPIRE` 就会延长别人的锁。

续期也要 Lua：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("PEXPIRE", KEYS[1], ARGV[2])
else
    return 0
end
```

含义：

```text
只有锁 value 仍然等于我的 request_id，才续期。
```

---

## 六、Go 续期脚本

```go
var renewLockScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("PEXPIRE", KEYS[1], ARGV[2])
else
    return 0
end
`)
```

续期函数：

```go
func (l *RedisLock) Renew(ctx context.Context, lock *Lock) (bool, error) {
    ttlMillis := strconv.FormatInt(lock.TTL.Milliseconds(), 10)

    n, err := renewLockScript.Run(ctx, l.rdb, []string{lock.Key}, lock.Value, ttlMillis).Int()
    if err != nil {
        return false, err
    }
    return n == 1, nil
}
```

使用 `PEXPIRE` 是为了支持毫秒级 TTL。

---

## 七、Watch Dog 示例

```go
func (l *RedisLock) StartWatchDog(ctx context.Context, lock *Lock, logger *log.Logger) context.CancelFunc {
    dogCtx, cancel := context.WithCancel(ctx)
    interval := lock.TTL / 3

    go func() {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()

        for {
            select {
            case <-dogCtx.Done():
                return
            case <-ticker.C:
                renewCtx, renewCancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
                ok, err := l.Renew(renewCtx, lock)
                renewCancel()

                if err != nil {
                    logger.Printf("renew lock failed key=%s err=%v", lock.Key, err)
                    continue
                }
                if !ok {
                    logger.Printf("renew lock skipped key=%s reason=not_owner_or_expired", lock.Key)
                    return
                }
            }
        }
    }()

    return cancel
}
```

业务完成时：

```go
stopDog := locker.StartWatchDog(ctx, lock, logger)
defer stopDog()
defer unlock()
```

注意：实际项目要更严谨地处理 goroutine 生命周期和日志指标。

---

## 八、Watch Dog 也不是银弹

续期机制会让锁更稳，但它仍然有风险：

- 客户端进程 STW 或卡死，续期 goroutine 不运行。
- 网络抖动导致续期失败。
- Redis 主从切换导致锁状态变化。
- 业务代码卡住，但 Watch Dog 仍然续期，锁被长时间占用。
- 请求已经没有意义，但锁还在续。

所以 Watch Dog 不是“开了就安全”。

它只是降低正常长任务误过期的概率。

---

## 九、什么时候不建议续期

不建议依赖续期的场景：

- 用户请求链路很短，应该快速完成。
- 业务强一致要求很高。
- 任务可能卡死很久。
- 无法接受锁长期占用。
- 后果严重，必须由数据库约束兜底。

对于长任务，更推荐：

- 拆成小任务。
- 用任务状态机。
- 用数据库行状态抢占。
- 用消息队列和消费幂等。
- 用任务心跳和超时回收。

不要把一个复杂长流程全靠 Redis 锁包起来。

---

## 十、锁过期后的防线

即使有 Watch Dog，也要假设锁可能失效。

业务层要有额外防线：

- 数据库唯一约束防重复创建。
- 订单状态条件更新。
- 乐观锁 version。
- fencing token。
- 幂等请求表。
- 操作日志和补偿任务。

分布式锁是第一道门，不是最后一道门。

---

## 十一、常见错误

### 1. TTL 随便设置

没有基于业务耗时和监控数据调整。

### 2. 续期不校验 value

可能给别人的锁续期。

### 3. Watch Dog 不停止

业务完成后 goroutine 继续运行，造成泄漏。

### 4. 长任务完全依赖 Redis 锁

长任务更适合状态机、消息队列和数据库抢占。

### 5. 锁过期后没有业务防线

旧请求可能继续写入数据库。

---

## 十二、本节练习

请完成下面练习：

1. 画出业务超过锁 TTL 后的并发时间线。
2. 根据一个 800ms P99 的业务估算锁 TTL。
3. 写出安全续期 Lua 脚本。
4. 在 Go 中封装 `Renew` 方法。
5. 设计 Watch Dog 的续期间隔。
6. 思考 Watch Dog 什么时候应该停止。
7. 思考业务卡死但 Watch Dog 仍续期会发生什么。
8. 为一个长任务设计不用 Redis 锁的替代方案。

---

## 十三、本节小结

这一节你学习了锁 TTL 和自动续期。

你需要记住：

- 锁必须有 TTL，但 TTL 过短会导致业务没执行完锁就过期。
- 锁 TTL 应该根据业务耗时和监控数据估算。
- Watch Dog 可以自动续期，降低误过期风险。
- 续期必须校验锁 value。
- Watch Dog 不能代替幂等、数据库约束和状态机。
- 长任务不要轻易用一把 Redis 锁从头锁到尾。

下一节我们会讨论业务幂等：为什么锁不能替代幂等。

