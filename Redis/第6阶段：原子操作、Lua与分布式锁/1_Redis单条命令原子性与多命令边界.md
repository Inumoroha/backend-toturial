# 1. Redis 单条命令原子性与多命令边界

前面阶段已经多次提到：Redis 是单线程处理命令的，单条命令天然具有原子性。

但这句话很容易被误解。它不等于“只要用了 Redis，并发问题就自动消失”。Redis 只能保证一条命令在执行时不会被其他命令插进来，多条命令组合起来并不自动原子。

学完这一节后，你应该能够：

- 理解 Redis 单条命令原子性的含义。
- 区分单命令原子和业务流程原子。
- 判断哪些操作可以直接依赖 Redis 命令。
- 理解多命令并发交错带来的问题。
- 知道什么时候需要 Lua、事务、数据库约束或业务幂等。

---

## 一、什么是单条命令原子性

原子性可以简单理解为：

```text
一个操作要么完整执行，要么不执行；
执行过程中不会被其他操作插入到中间。
```

Redis 执行单条命令时，其他客户端的命令不会插入这条命令内部。

例如：

```redis
INCR article:view_count:1001
```

即使有很多客户端同时执行 `INCR`，Redis 也会一条一条处理。

最终计数不会因为并发写入而丢失。

---

## 二、哪些单条命令可以放心依赖

常见的原子命令：

```redis
INCR counter:user:1001
DECR stock:sku:2001
HINCRBY article:1001 view_count 1
SADD user:tags:1001 redis
ZINCRBY article:rank 1 1001
SET lock:order:1001 request-id NX EX 10
```

这些命令的内部逻辑由 Redis 一次完成。

例如 `SADD` 用来去重：

```redis
SADD lottery:users 1001
SADD lottery:users 1001
```

同一个 member 重复加入，集合里也只会有一份。

---

## 三、多条命令不自动原子

看一个常见库存逻辑：

```text
1. GET stock:sku:2001
2. 判断库存是否大于 0
3. DECR stock:sku:2001
```

如果两个请求同时执行，可能出现下面的交错：

```text
库存 = 1

请求 A：GET stock -> 1
请求 B：GET stock -> 1
请求 A：判断可以扣减
请求 B：判断可以扣减
请求 A：DECR -> 0
请求 B：DECR -> -1
```

单条 `GET` 是原子的，单条 `DECR` 也是原子的。

但 `GET + 判断 + DECR` 这三个步骤组合起来不是原子的。

---

## 四、再看释放锁的多命令问题

释放分布式锁时，很多人会先写成：

```redis
GET lock:order:1001
DEL lock:order:1001
```

业务含义是：

```text
如果锁 value 是我的 request_id，就删除锁。
```

但如果 `GET` 和 `DEL` 分开执行，就会有问题：

```text
请求 A 持有锁，value=A，过期时间 10s
请求 A 执行业务超过 10s，锁自动过期
请求 B 获取同一把锁，value=B
请求 A 执行 DEL lock:order:1001
```

如果 A 不校验 value 或校验和删除不是原子的，就可能删掉 B 的锁。

所以释放锁通常要用 Lua：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

---

## 五、Redis 事务能不能解决

Redis 有 `MULTI`、`EXEC`、`WATCH`。

它们可以解决一部分多命令组合问题，但要注意它们和关系型数据库事务不一样。

基本事务：

```redis
MULTI
INCR counter:a
INCR counter:b
EXEC
```

`EXEC` 执行时，队列里的命令会连续执行。

但 Redis 事务没有自动回滚。某条命令运行失败，其他命令可能已经执行。

如果需要“读后判断再写”，通常要配合 `WATCH`：

```redis
WATCH stock:sku:2001
GET stock:sku:2001
MULTI
DECR stock:sku:2001
EXEC
```

`WATCH` 能做乐观锁，但代码复杂度较高。

在很多 Redis 原子小逻辑里，Lua 更直接。

---

## 六、Lua 解决的边界

Lua 可以把多步 Redis 命令合成一个脚本，在 Redis 内部一次执行。

适合：

- 判断 value 后删除 key。
- 判断库存后扣减。
- 限流时初始化 TTL 并自增。
- 检查 Set 是否存在成员后写入。
- 多个小 key 的一致更新。

不适合：

- 长时间计算。
- 大量循环扫描。
- 访问数据库或外部服务。
- 承担完整业务事务。
- 替代业务幂等和数据库约束。

Lua 保证的是 Redis 内部脚本执行的原子性。

它不能保证 Redis 和 MySQL 之间天然一致。

---

## 七、业务流程原子性不只靠 Redis

真实后端业务往往长这样：

```text
1. 校验用户
2. 查商品
3. 扣 Redis 库存
4. 创建数据库订单
5. 支付
6. 发消息
```

Redis 只能保护其中一小段。

如果第 3 步成功，第 4 步失败，就需要补偿。

如果用户重复提交，就需要幂等。

如果 Redis 锁过期后业务仍在执行，就可能出现并发写入。

所以完整方案通常还需要：

- 数据库唯一约束。
- 乐观锁版本号。
- 业务幂等表。
- 消息队列补偿。
- 状态机约束。
- 监控和人工处理入口。

---

## 八、如何判断是否需要 Lua 或锁

先问三个问题：

```text
1. 这个逻辑能不能用一条 Redis 命令完成？
2. 如果不能，多条 Redis 命令之间是否有并发交错风险？
3. 如果 Redis 这一步成功，但后续数据库失败，业务怎么补偿？
```

示例：

| 场景 | 推荐方式 |
| --- | --- |
| 浏览量 +1 | `INCR` |
| 用户标签去重 | `SADD` |
| 排行榜加分 | `ZINCRBY` |
| 判断库存后扣减 | Lua |
| 释放分布式锁 | Lua |
| 防止重复下单 | 数据库唯一约束 + 幂等 |
| 订单状态流转 | 数据库事务或状态机 |

不要一看到并发就上分布式锁。

先看有没有更简单、更可靠的约束方式。

---

## 九、Go 里容易犯的错

错误写法：

```go
stock, err := rdb.Get(ctx, key).Int()
if err != nil {
    return err
}

if stock <= 0 {
    return ErrStockNotEnough
}

if err := rdb.Decr(ctx, key).Err(); err != nil {
    return err
}
```

这段代码在低并发下看起来没问题。

但 `GET` 和 `DECR` 之间可能插入其他请求。

更合理的做法是用 Lua：

```go
var deductScript = redis.NewScript(`
local stock = tonumber(redis.call("GET", KEYS[1]))
if stock == nil then
    return -1
end
if stock <= 0 then
    return -2
end
return redis.call("DECR", KEYS[1])
`)
```

---

## 十、常见错误

### 1. 把单命令原子理解成业务全流程原子

Redis 只能保证单条命令或 Lua 脚本在 Redis 内部原子执行。

### 2. 用 `GET` 后再 `SET` 处理并发状态

读后判断再写，通常要用 Lua、`WATCH` 或数据库约束。

### 3. 以为加锁后就不需要幂等

锁可能超时、失败或被绕过。幂等仍然是业务必须考虑的。

### 4. 用复杂 Lua 写完整业务

Lua 要短小，不要把数据库事务、网络调用、复杂计算塞进 Redis。

### 5. 忽略后续补偿

Redis 扣减成功不代表订单一定创建成功，失败后要有补偿策略。

---

## 十一、本节练习

请完成下面练习：

1. 举出 5 个 Redis 单条原子命令。
2. 解释为什么 `GET + DECR` 不是原子的。
3. 写出释放锁时 `GET + DEL` 的并发问题。
4. 思考 `INCR` 为什么适合做计数器。
5. 用 Lua 写一个判断库存大于 0 后扣减的脚本。
6. 思考 Redis Lua 能否保证 Redis 和数据库一致。
7. 判断“防止重复下单”应该主要靠 Redis 锁还是数据库唯一约束。
8. 总结你项目里哪些操作可以用 Redis 单命令完成。

---

## 十二、本节小结

这一节你学习了 Redis 原子性的边界。

你需要记住：

- Redis 单条命令是原子的。
- 多条命令组合起来不自动原子。
- `GET + 判断 + SET/DEL/DECR` 是并发问题高发区。
- Lua 可以把短小的多步 Redis 逻辑变成原子脚本。
- Redis 原子性不能替代数据库事务、业务幂等和补偿机制。

下一节开始，我们会进入分布式锁的基础实现：`SET key value NX EX seconds`。

