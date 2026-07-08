# 8. Lua 脚本：Eval 与 EvalSha

Redis 单条命令是原子的，但多条命令组合起来不自动原子。

比如释放分布式锁时，你需要先判断锁是不是自己持有的，再删除锁。如果这两步分开执行，就可能出现并发问题。Lua 脚本可以把多步逻辑放到 Redis 里一次执行，从而保证脚本执行期间的原子性。

学完这一节后，你应该能够：

- 理解 Redis Lua 脚本解决什么问题。
- 使用 `Eval` 执行 Lua 脚本。
- 使用 `EvalSha` 执行已加载脚本。
- 在 Lua 中使用 `KEYS` 和 `ARGV`。
- 用 Lua 实现安全释放锁。
- 用 Lua 实现简单扣库存原子逻辑。
- 知道 Lua 脚本的适用场景和限制。

---

## 一、为什么需要 Lua

假设释放锁流程是：

```text
1. GET lock:key，判断 value 是否等于自己的 request_id
2. 如果相等，DEL lock:key
```

如果用两条命令：

```redis
GET lock:order:1001
DEL lock:order:1001
```

中间可能发生锁过期并被别人重新获取的情况。

你可能误删别人的锁。

Lua 可以把判断和删除放到一个脚本里，让 Redis 原子执行。

---

## 二、Lua 脚本的基本结构

一个简单脚本：

```lua
return redis.call("GET", KEYS[1])
```

含义：读取第一个 key。

在 Go 中执行：

```go
script := `return redis.call("GET", KEYS[1])`

val, err := rdb.Eval(ctx, script, []string{"name"}).Result()
if err != nil {
    return err
}
fmt.Println(val)
```

`KEYS` 用来传 key。

`ARGV` 用来传普通参数。

---

## 三、KEYS 和 ARGV

Redis Lua 脚本中：

```lua
KEYS[1]
KEYS[2]
ARGV[1]
ARGV[2]
```

注意：Lua 数组下标从 1 开始。

Go 调用：

```go
script := `
return {KEYS[1], KEYS[2], ARGV[1], ARGV[2]}
`

val, err := rdb.Eval(ctx, script,
    []string{"key1", "key2"},
    "arg1", "arg2",
).Result()
```

约定：

- key 放到 `KEYS`。
- 非 key 参数放到 `ARGV`。

不要把 key 放到 `ARGV` 里，这在 Redis Cluster 场景会有问题。

---

## 四、Eval 基本用法

`Eval` 每次都把脚本文本发送给 Redis。

```go
result, err := rdb.Eval(ctx, script, keys, args...).Result()
```

参数：

| 参数 | 含义 |
| --- | --- |
| `ctx` | context |
| `script` | Lua 脚本文本 |
| `keys` | key 列表，对应 Lua 中的 `KEYS` |
| `args` | 参数列表，对应 Lua 中的 `ARGV` |

示例：判断 key 值是否等于指定参数：

```go
script := `
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return 1
else
    return 0
end
`

n, err := rdb.Eval(ctx, script, []string{"name"}, "redis").Int()
```

`Int()` 可以把结果转成 int。

---

## 五、安全释放锁脚本

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
如果锁 value 等于我的 request_id，就删除锁
否则不删除
```

Go 代码：

```go
var releaseLockScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
`)

func ReleaseLock(ctx context.Context, rdb *redis.Client, key, requestID string) error {
    n, err := releaseLockScript.Run(ctx, rdb, []string{key}, requestID).Int()
    if err != nil {
        return err
    }
    if n == 0 {
        return errors.New("lock not owned or already expired")
    }
    return nil
}
```

`redis.NewScript` 会帮助管理脚本加载和执行，使用起来比手动 `EvalSha` 更方便。

---

## 六、为什么释放锁必须校验 value

如果不校验 value，可能误删别人的锁。

场景：

```text
线程 A 获取锁，value=A，过期时间 10s
线程 A 执行业务超过 10s，锁过期
线程 B 获取锁，value=B
线程 A 执行 DEL lock:key
```

如果 A 直接删除，它会删掉 B 的锁。

所以释放锁必须判断：

```text
锁的 value 仍然等于我的 request_id
```

Lua 保证判断和删除是一个原子动作。

---

## 七、简单扣库存脚本

需求：库存大于 0 时扣减，否则返回库存不足。

Lua：

```lua
local stock = tonumber(redis.call("GET", KEYS[1]))
if stock == nil then
    return -1
end
if stock <= 0 then
    return 0
end
redis.call("DECR", KEYS[1])
return stock - 1
```

返回含义：

| 返回值 | 含义 |
| --- | --- |
| `-1` | 库存 key 不存在 |
| `0` | 库存不足，或扣完后为 0 要结合业务解释 |
| 正数 | 扣减后的库存 |

这个脚本只是教学示例。

真实库存系统还要考虑数据库一致性、订单超时释放、幂等、补偿等问题。

---

## 八、Go 执行扣库存脚本

```go
var deductStockScript = redis.NewScript(`
local stock = tonumber(redis.call("GET", KEYS[1]))
if stock == nil then
    return -1
end
if stock <= 0 then
    return -2
end
redis.call("DECR", KEYS[1])
return stock - 1
`)

func DeductStock(ctx context.Context, rdb *redis.Client, productID int64) (int, error) {
    key := fmt.Sprintf("stock:product:%d", productID)

    n, err := deductStockScript.Run(ctx, rdb, []string{key}).Int()
    if err != nil {
        return 0, err
    }

    switch n {
    case -1:
        return 0, errors.New("stock key not found")
    case -2:
        return 0, errors.New("stock not enough")
    default:
        return n, nil
    }
}
```

这里用 `-2` 表示库存不足，避免和扣减后库存为 0 混淆。

---

## 九、EvalSha 是什么

`Eval` 每次发送完整脚本文本。

`EvalSha` 是先把脚本加载到 Redis，之后通过脚本 SHA1 执行。

流程：

```text
SCRIPT LOAD -> 得到 sha
EVALSHA sha -> 执行脚本
```

Go 手动写法：

```go
sha, err := rdb.ScriptLoad(ctx, script).Result()
if err != nil {
    return err
}

result, err := rdb.EvalSha(ctx, sha, []string{key}, args...).Result()
```

不过实际项目里更推荐使用：

```go
redis.NewScript(script).Run(ctx, rdb, keys, args...)
```

它会自动处理脚本加载和执行。

---

## 十、Lua 脚本的限制

Lua 脚本要谨慎使用。

注意：

- 脚本执行期间会阻塞 Redis 主线程处理其他命令。
- 脚本不能写得太复杂。
- 不要在 Lua 里做大量循环。
- 不要处理超大 key。
- 脚本要保持短小、确定、可测试。
- Redis Cluster 中 key 要符合同槽要求。

Lua 适合小而关键的原子逻辑，不适合大段业务代码。

---

## 十一、Lua 适合什么场景

适合：

- 安全释放锁。
- 判断库存再扣减。
- 限流窗口原子初始化和自增。
- 多 key 小范围原子更新。
- 检查 value 后删除或更新。

不适合：

- 复杂业务流程。
- 大量数据扫描。
- 长时间计算。
- 需要访问外部服务。
- 替代数据库事务。

---

## 十二、常见错误

### 1. key 放到 ARGV

Redis Cluster 场景会出问题。key 应该放到 `KEYS`。

### 2. Lua 脚本太长太复杂

会阻塞 Redis，难以维护。

### 3. 释放锁不校验 value

可能误删别人的锁。

### 4. 返回值语义不清晰

脚本返回 `0`、`-1`、`1` 时，要在代码里明确含义。

### 5. 以为 Lua 能解决所有一致性问题

Lua 只能保证 Redis 内部脚本原子执行，不能自动保证 Redis 和数据库一致。

---

## 十三、本节练习

请完成下面练习：

1. 写一个 Lua 脚本读取 `KEYS[1]`。
2. 使用 `Eval` 在 Go 中执行它。
3. 写一个判断 key 值是否等于参数的脚本。
4. 写安全释放锁脚本。
5. 使用 `redis.NewScript` 封装释放锁脚本。
6. 写简单扣库存脚本。
7. 设计扣库存脚本的返回值语义。
8. 使用 `ScriptLoad` 和 `EvalSha` 手动执行脚本。
9. 思考为什么 Lua 脚本不能写太复杂。
10. 思考 Lua 能否解决 Redis 和数据库的一致性问题。

---

## 十四、本节小结

这一节你学习了 Redis Lua 脚本在 Go 中的使用。

你需要记住：

- Lua 用来把多步 Redis 逻辑合成一个原子脚本。
- `Eval` 直接执行脚本文本。
- `EvalSha` 通过脚本 SHA 执行已加载脚本。
- `redis.NewScript` 是 Go 中更方便的封装。
- key 放到 `KEYS`，普通参数放到 `ARGV`。
- 安全释放锁必须校验 value 后再删除。
- Lua 适合短小关键的原子逻辑，不适合复杂业务。

下一节我们会把前面所有内容组合起来，写一个完整的文章缓存模块：`GetArticle`、`SetArticleCache`、`DeleteArticleCache`。
