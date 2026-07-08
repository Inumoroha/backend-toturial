# 2. SET NX EX 实现基础分布式锁

分布式锁的目标是：在多个进程、多个服务实例之间，让同一时间只有一个执行者进入某段临界区。

Redis 最常见的基础锁写法是：

```redis
SET lock:order:1001 request-id NX EX 10
```

这条命令同时完成：

- key 不存在时才写入。
- 写入锁的持有者标识。
- 设置过期时间。

学完这一节后，你应该能够：

- 理解分布式锁解决什么问题。
- 使用 `SET key value NX EX seconds` 获取锁。
- 理解为什么不能用 `SETNX` 后再 `EXPIRE`。
- 在 Go 中封装基础加锁方法。
- 知道基础锁的适用场景和明显缺陷。

---

## 一、什么时候需要分布式锁

单机程序里可以用互斥锁：

```go
var mu sync.Mutex
```

但如果服务部署了多个实例：

```text
api-1
api-2
api-3
```

每个实例都有自己的内存锁。`sync.Mutex` 只能保护当前进程，保护不了其他机器上的进程。

这时才可能需要分布式锁。

典型场景：

- 防止同一订单被多个实例同时处理。
- 防止同一热点缓存被多个实例同时重建。
- 控制同一任务同一时间只被一个 worker 执行。
- 避免某些低频管理操作重复运行。

---

## 二、锁的基本语义

一把锁通常需要满足：

```text
1. 互斥：同一时间只有一个持有者。
2. 自动过期：持有者崩溃后锁最终能释放。
3. 安全释放：只能释放自己持有的锁。
4. 有限等待：获取不到锁时不能无限卡住请求。
```

Redis 基础锁主要靠这条命令：

```redis
SET lock:resource request-id NX EX 10
```

含义：

| 参数 | 含义 |
| --- | --- |
| `lock:resource` | 锁 key |
| `request-id` | 锁 value，标识持有者 |
| `NX` | key 不存在时才设置 |
| `EX 10` | 10 秒后自动过期 |

---

## 三、为什么不用 SETNX 后再 EXPIRE

错误写法：

```redis
SETNX lock:order:1001 request-id
EXPIRE lock:order:1001 10
```

这两条命令之间不是原子的。

如果执行完 `SETNX` 后，服务崩溃了，`EXPIRE` 没执行：

```text
锁 key 永久存在
其他请求永远拿不到锁
```

正确写法是把加锁和过期时间放在同一条 `SET` 命令里：

```redis
SET lock:order:1001 request-id NX EX 10
```

这样加锁和设置过期时间是一个原子操作。

---

## 四、锁 key 怎么设计

锁 key 要表达被保护的资源。

示例：

```text
lock:order:1001
lock:stock:sku:2001
lock:cache_rebuild:article:9527
lock:job:daily_report
```

不要所有业务共用一把大锁：

```text
lock:global
```

大锁会降低并发能力。

应该尽量锁住最小资源范围：

```text
处理订单 1001，只锁 order:1001
重建文章 9527，只锁 article:9527
```

---

## 五、锁 value 为什么要唯一

锁 value 不能随便写成固定值：

```redis
SET lock:order:1001 locked NX EX 10
```

更推荐写成唯一标识：

```text
request-id
uuid
instance-id + goroutine-id + timestamp
```

原因是释放锁时要确认：

```text
这把锁仍然是我持有的
```

如果 value 不唯一，就很难判断锁归属。

下一节会专门讲误删别人的锁问题。

---

## 六、Go 获取锁基础写法

使用 `go-redis/v9`：

```go
func TryLock(ctx context.Context, rdb *redis.Client, key, value string, ttl time.Duration) (bool, error) {
    return rdb.SetNX(ctx, key, value, ttl).Result()
}
```

`SetNX` 在 go-redis 中会生成带过期时间的原子命令。

调用示例：

```go
requestID := uuid.NewString()
locked, err := TryLock(ctx, rdb, "lock:order:1001", requestID, 10*time.Second)
if err != nil {
    return err
}
if !locked {
    return errors.New("resource is busy")
}
```

拿不到锁时，不要直接长时间阻塞请求。

可以返回“稍后再试”，也可以短暂重试。

---

## 七、带超时的加锁 context

获取锁是 Redis 操作，也要有超时。

```go
lockCtx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
defer cancel()

locked, err := rdb.SetNX(lockCtx, key, value, 10*time.Second).Result()
```

不要让一个获取锁操作卡住整个接口。

接口总超时是 1 秒时，获取 Redis 锁可能只应该给几十到一百多毫秒。

具体数值要结合部署环境和业务容忍度。

---

## 八、获取不到锁怎么办

常见处理方式：

| 场景 | 处理方式 |
| --- | --- |
| 用户重复点击 | 直接返回处理中 |
| 后台任务重复执行 | 跳过本轮 |
| 热点缓存重建 | 返回旧缓存 |
| 秒杀扣库存 | 返回稍后重试或排队 |
| 管理后台操作 | 提示已有任务运行中 |

不要所有场景都死等。

死等会占住请求线程、连接和上下文资源。

---

## 九、简单重试

有些场景可以短暂重试：

```go
func TryLockWithRetry(ctx context.Context, rdb *redis.Client, key, value string, ttl time.Duration) (bool, error) {
    ticker := time.NewTicker(50 * time.Millisecond)
    defer ticker.Stop()

    deadline := time.After(300 * time.Millisecond)

    for {
        locked, err := rdb.SetNX(ctx, key, value, ttl).Result()
        if err != nil {
            return false, err
        }
        if locked {
            return true, nil
        }

        select {
        case <-ctx.Done():
            return false, ctx.Err()
        case <-deadline:
            return false, nil
        case <-ticker.C:
        }
    }
}
```

重试时间要短。

重试间隔可以增加随机抖动，避免大量请求同一时间再次竞争。

---

## 十、基础锁的完整流程

```text
1. 生成唯一 request_id
2. SET lock_key request_id NX EX ttl
3. 如果失败，按业务策略返回或重试
4. 如果成功，执行业务逻辑
5. 释放锁时校验 value 后删除
```

注意第 5 步不能简单 `DEL`。

释放锁必须安全释放。

后面会用 Lua 实现。

---

## 十一、基础锁的缺陷

`SET NX EX` 是基础，不是完整答案。

它有几个明显问题：

- 业务执行时间可能超过锁 TTL。
- 锁过期后，其他请求可以拿到锁。
- 原持有者可能继续执行并写数据库。
- Redis 主从切换时可能丢锁。
- 客户端阻塞、GC、网络抖动都会影响锁语义。
- 锁不能代替业务幂等。

所以分布式锁必须谨慎使用。

如果业务后果很严重，还要考虑 fencing token、数据库版本号、唯一约束等机制。

---

## 十二、常见错误

### 1. `SETNX` 后再 `EXPIRE`

两条命令之间服务可能崩溃，导致死锁。

### 2. 锁没有过期时间

持有者崩溃后锁无法自动释放。

### 3. 所有资源共用一把锁

会把系统并发能力压得很低。

### 4. 锁 value 固定

释放锁时无法准确判断锁归属。

### 5. 获取锁无超时

请求可能被 Redis 抖动拖住。

---

## 十三、本节练习

请完成下面练习：

1. 用 Redis 命令获取一把 `lock:order:1001` 锁。
2. 给锁设置 10 秒过期时间。
3. 解释为什么不能使用 `SETNX` 后再 `EXPIRE`。
4. 为文章缓存重建设计锁 key。
5. 在 Go 中封装 `TryLock`。
6. 给加锁操作增加 100ms 超时。
7. 思考获取不到锁时，接口应该返回什么。
8. 思考锁 TTL 应该如何估算。

---

## 十四、本节小结

这一节你学习了 Redis 分布式锁的基础获取方式。

你需要记住：

- 基础加锁命令是 `SET key value NX EX seconds`。
- 加锁和设置过期时间必须在同一条命令里完成。
- 锁 key 要锁住最小资源范围。
- 锁 value 要使用唯一标识。
- 获取锁要有超时，拿不到锁要有业务处理策略。
- 基础锁还有很多风险，后面几节会逐步补齐。

下一节我们重点讲锁 value 的唯一性，以及为什么释放锁不能直接 `DEL`。

