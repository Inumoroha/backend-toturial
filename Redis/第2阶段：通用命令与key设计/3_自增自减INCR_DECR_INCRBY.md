# 3. 自增自减：INCR、DECR、INCRBY

Redis 很适合做计数器。文章阅读量、点赞数、接口调用次数、库存预扣、限流窗口计数，都可能用到自增自减命令。

这一节学习 `INCR`、`DECR`、`INCRBY`。

学完这一节后，你应该能够：

- 使用 `INCR` 对数字自增 1。
- 使用 `DECR` 对数字自减 1。
- 使用 `INCRBY` 按指定步长增加。
- 理解 Redis 自增命令的原子性。
- 知道计数器和限流计数的基本设计方式。
- 避免把非数字 value 当作计数器操作。

---

## 一、为什么 Redis 适合做计数器

计数器通常有两个特点：

1. 操作频繁。
2. 每次只做简单加减。

比如：

- 文章阅读量 +1。
- 视频点赞数 +1。
- 接口访问次数 +1。
- 短信发送次数 +1。
- 库存预扣 -1。

Redis 的自增自减命令是单条命令，执行速度快，并且具有原子性。多个客户端同时执行 `INCR`，Redis 会一个一个处理，不会出现并发覆盖的问题。

---

## 二、INCR：自增 1

`INCR` 用来把 key 对应的数字加 1。

语法：

```redis
INCR key
```

示例：

```redis
SET article:view_count:1001 0
INCR article:view_count:1001
```

返回：

```text
(integer) 1
```

再次执行：

```redis
INCR article:view_count:1001
```

返回：

```text
(integer) 2
```

读取：

```redis
GET article:view_count:1001
```

返回：

```text
"2"
```

注意：Redis String 可以保存数字。`INCR` 操作的就是 String 类型中可以被解析成整数的 value。

---

## 三、key 不存在时 INCR 会自动创建

如果 key 不存在：

```redis
DEL counter
INCR counter
```

返回：

```text
(integer) 1
```

Redis 会把不存在的 key 当作 0，然后加 1。

这对计数器很方便。

比如文章第一次被阅读时，不需要先判断 key 是否存在：

```redis
INCR article:view_count:1001
```

如果没有这个 key，它会自动从 1 开始。

---

## 四、DECR：自减 1

`DECR` 用来把 key 对应的数字减 1。

语法：

```redis
DECR key
```

示例：

```redis
SET stock:product:1001 10
DECR stock:product:1001
```

返回：

```text
(integer) 9
```

再次执行：

```redis
DECR stock:product:1001
```

返回：

```text
(integer) 8
```

如果 key 不存在，`DECR` 会把它当作 0，然后减 1：

```redis
DEL stock:test
DECR stock:test
```

返回：

```text
(integer) -1
```

所以用 `DECR` 做库存时要小心，不能只依赖一个简单命令，否则可能减成负数。后面学习 Lua 时会讲更严谨的扣库存方式。

---

## 五、INCRBY：按指定数值增加

`INCRBY` 用来让 key 增加指定整数。

语法：

```redis
INCRBY key increment
```

示例：

```redis
SET article:view_count:1001 10
INCRBY article:view_count:1001 5
```

返回：

```text
(integer) 15
```

`INCRBY` 也可以传负数：

```redis
INCRBY article:view_count:1001 -3
```

返回：

```text
(integer) 12
```

不过为了可读性，减少通常使用 `DECR` 或 `DECRBY`。

Redis 也有 `DECRBY`：

```redis
DECRBY stock:product:1001 2
```

虽然本阶段重点列的是 `INCRBY`，但你知道 `DECRBY` 存在即可。

---

## 六、value 必须是整数

如果 value 不是整数，执行 `INCR` 会报错。

```redis
SET name redis
INCR name
```

返回类似：

```text
ERR value is not an integer or out of range
```

如果 value 是字符串形式的整数，则可以：

```redis
SET count "10"
INCR count
```

返回：

```text
(integer) 11
```

所以计数器 key 要保持纯数字，不要把 JSON、文本和计数混在一个 key 里。

---

## 七、自增命令的原子性

`INCR` 是单条 Redis 命令，具有原子性。

假设很多请求同时执行：

```redis
INCR article:view_count:1001
```

Redis 会按顺序处理每个请求：

```text
0 -> 1
1 -> 2
2 -> 3
3 -> 4
```

不会出现两个请求都读到 0，然后都写回 1 的问题。

这就是 Redis 适合做高频计数器的原因之一。

但要注意：多个命令组合不自动原子。

比如：

```redis
GET stock:product:1001
DECR stock:product:1001
```

这两步之间可能插入其他请求。复杂判断加更新需要 Lua、事务或数据库约束来保证。

---

## 八、业务场景：文章阅读量

文章每被访问一次，阅读量加 1：

```redis
INCR article:view_count:1001
```

读取阅读量：

```redis
GET article:view_count:1001
```

如果文章很多，可以按文章 ID 设计 key：

```text
article:view_count:1001
article:view_count:1002
article:view_count:1003
```

如果阅读量最终要保存到数据库，常见做法是：

1. Redis 承接高频自增。
2. 定时任务批量同步到数据库。
3. 数据库保存最终统计结果。

这样可以减少数据库频繁更新压力。

---

## 九、业务场景：固定窗口限流

假设要限制某个 IP 每分钟最多访问登录接口 10 次。

key 可以设计为：

```text
rate_limit:login:ip:127.0.0.1:202606271530
```

第一次访问：

```redis
INCR rate_limit:login:ip:127.0.0.1:202606271530
EXPIRE rate_limit:login:ip:127.0.0.1:202606271530 60
```

每次访问都 `INCR`，如果结果大于 10，就拒绝。

不过这里有一个细节：`INCR` 和 `EXPIRE` 是两条命令，中间可能出现异常。

更严谨的做法可以用 Lua 脚本保证第一次自增和设置过期时间一起完成。

当前阶段先理解固定窗口限流的基本思路：

```text
同一个窗口内，对同一个 key 计数
超过阈值就拒绝
窗口过期后重新计数
```

---

## 十、业务场景：库存预扣

简单库存可以这样：

```redis
SET stock:product:1001 100
DECR stock:product:1001
```

但真实库存不能只这么写。

原因是：

- 可能减成负数。
- Redis 和数据库库存要一致。
- 订单取消要回补。
- 支付超时要释放。
- 高并发下需要严谨的原子判断。

更安全的扣库存逻辑通常是：

```text
判断库存是否大于 0
如果大于 0，再扣减
否则返回库存不足
```

这需要 Lua 脚本或数据库事务配合。

所以你现在可以知道：`DECR` 可以表达扣减动作，但真实库存系统不能只靠一个 `DECR` 草草结束。

---

## 十一、计数器是否需要 TTL

有些计数器需要 TTL，有些不需要。

| 场景 | 是否需要 TTL | 原因 |
| --- | --- | --- |
| 文章总阅读量 | 通常不需要，或长期保留 | 是累计统计 |
| 每分钟接口访问次数 | 需要 | 窗口结束后应自动清理 |
| 每天用户签到次数 | 可能需要 | 按天统计可以设置过期 |
| 短信发送次数限制 | 需要 | 限制窗口结束后重置 |
| 商品库存 | 通常不靠 TTL | 库存是业务状态 |

设计计数器 key 时，一定要一起思考 TTL。

---

## 十二、常见错误

### 1. 对非数字 value 执行 INCR

```redis
SET count hello
INCR count
```

会报错。

### 2. 以为 INCRBY 只能增加

`INCRBY key -5` 可以减少，但可读性不一定好。

### 3. 忘记给限流计数设置 TTL

限流 key 如果不设置 TTL，会一直存在，用户可能永远被限制。

### 4. 用简单 DECR 做真实库存

真实库存涉及一致性和业务补偿，不能只靠一个 `DECR`。

---

## 十三、本节练习

请完成下面练习：

1. 写入 `counter` 为 0。
2. 连续执行 3 次 `INCR counter`。
3. 使用 `GET counter` 查看结果。
4. 执行 `INCRBY counter 10`。
5. 执行 `DECR counter`。
6. 对不存在的 key 执行 `INCR new_counter`，观察结果。
7. 写入 `article:view_count:1001`，模拟 5 次阅读。
8. 写入 `rate_limit:login:user:1001`，使用 `INCR` 和 `EXPIRE` 模拟 60 秒限流窗口。
9. 对非数字 value 执行 `INCR`，观察错误。
10. 思考：哪些计数器需要 TTL，哪些不需要？

---

## 十四、本节小结

这一节你掌握了 Redis 自增自减命令：

- `INCR key`：自增 1。
- `DECR key`：自减 1。
- `INCRBY key n`：增加指定整数。
- key 不存在时，`INCR` 会从 0 开始。
- value 必须能解析成整数。
- 单条自增自减命令具有原子性。
- 计数器常用于阅读量、点赞数、限流、短信次数等场景。

下一节我们会学习批量读取和写入：`MGET`、`MSET`。它们能减少多次网络往返，让缓存读取更高效。
