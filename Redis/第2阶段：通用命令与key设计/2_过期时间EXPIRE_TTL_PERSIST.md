# 2. 过期时间：EXPIRE、TTL、PERSIST

Redis 很适合保存临时数据，而临时数据最重要的能力之一就是自动过期。

这一节专门学习 Redis 的过期时间。它会直接影响缓存、验证码、登录 token、限流计数和热点数据管理。

学完这一节后，你应该能够：

- 使用 `EXPIRE` 给 key 设置过期时间。
- 使用 `TTL` 查看 key 剩余生存时间。
- 使用 `PERSIST` 移除 key 的过期时间。
- 使用 `SET key value EX seconds` 写入时直接设置过期时间。
- 理解 TTL 是设置在整个 key 上的。
- 知道不同业务应该如何规划 TTL。

---

## 一、为什么需要过期时间

Redis 中很多数据不应该永久保存。

比如：

- 短信验证码 5 分钟后失效。
- 登录 token 7 天后失效。
- 文章详情缓存 30 分钟后刷新。
- 限流计数 1 分钟后重新开始。
- 防缓存穿透的空值缓存 1 分钟后过期。

如果这些数据不过期，就会带来几个问题：

- 内存持续增长。
- 用户可能使用过期验证码。
- 缓存长期不刷新，数据越来越旧。
- 限流计数无法自然重置。

所以 Redis 的 TTL 不是附加功能，而是缓存设计的核心能力。

---

## 二、EXPIRE：给 key 设置过期时间

`EXPIRE` 用来给一个已经存在的 key 设置过期时间，单位是秒。

语法：

```redis
EXPIRE key seconds
```

示例：

```redis
SET login:code:1001 888888
EXPIRE login:code:1001 300
```

表示 `login:code:1001` 会在 300 秒后过期。

返回值：

```text
(integer) 1
```

表示设置成功。

如果 key 不存在：

```redis
EXPIRE not_exists_key 300
```

返回：

```text
(integer) 0
```

表示没有设置成功，因为 key 不存在。

---

## 三、TTL：查看剩余过期时间

`TTL` 用来查看 key 还剩多少秒过期。

语法：

```redis
TTL key
```

示例：

```redis
SET login:code:1001 888888
EXPIRE login:code:1001 300
TTL login:code:1001
```

可能返回：

```text
(integer) 296
```

表示还剩 296 秒。

`TTL` 有几个特殊返回值：

| 返回值 | 含义 |
| --- | --- |
| 正整数 | key 剩余过期秒数 |
| `-1` | key 存在，但没有过期时间 |
| `-2` | key 不存在 |

示例：

```redis
SET name redis
TTL name
```

返回：

```text
(integer) -1
```

因为 `name` 存在，但没有设置过期时间。

删除后：

```redis
DEL name
TTL name
```

返回：

```text
(integer) -2
```

因为 key 不存在。

---

## 四、PERSIST：移除过期时间

`PERSIST` 用来移除 key 的过期时间，让它变成不过期。

语法：

```redis
PERSIST key
```

示例：

```redis
SET token abc
EXPIRE token 60
TTL token
PERSIST token
TTL token
```

执行 `PERSIST` 后，`TTL token` 返回：

```text
(integer) -1
```

表示 key 存在，但没有过期时间。

`PERSIST` 成功返回 `1`，如果 key 不存在或本来就没有过期时间，通常返回 `0`。

---

## 五、写入时直接设置过期时间

实际项目中，更推荐写入时直接设置过期时间。

```redis
SET login:code:1001 888888 EX 300
```

这条命令同时完成：

1. 写入 `login:code:1001`。
2. 设置 value 为 `888888`。
3. 设置 300 秒后过期。

它比下面两条命令更稳妥：

```redis
SET login:code:1001 888888
EXPIRE login:code:1001 300
```

因为两条命令之间可能出现异常。比如程序执行完 `SET` 后崩溃了，还没来得及执行 `EXPIRE`，这个验证码就可能永久存在。

所以写临时数据时，优先使用：

```redis
SET key value EX seconds
```

---

## 六、毫秒级过期时间

除了 `EX`，Redis 也支持毫秒级过期。

```redis
SET temp:value hello PX 5000
```

表示 5000 毫秒后过期，也就是 5 秒。

查看毫秒级剩余时间：

```redis
PTTL temp:value
```

初学阶段大多数场景用秒级 `EX` 和 `TTL` 就够了。毫秒级常见于更精细的锁、限流或临时状态。

---

## 七、TTL 是设置在整个 key 上的

这一点很容易被忽略。

对于 String：

```redis
SET article:detail:1001 "..."
EXPIRE article:detail:1001 1800
```

过期的是整个 `article:detail:1001`。

对于 Hash：

```redis
HSET user:profile:1001 name tom age 18
EXPIRE user:profile:1001 600
```

过期的也是整个 `user:profile:1001`，不是单独的 `name` 字段或 `age` 字段。

Redis 原生命令不能给 Hash 的某个字段单独设置 TTL。

如果你需要字段级过期，通常要换一种 key 设计，例如：

```text
user:profile:1001:name
user:profile:1001:age
```

或者在 value 中保存业务过期时间，由程序自己判断。

---

## 八、覆盖 key 会影响 TTL

这是一个非常重要的细节。

先写入并设置过期：

```redis
SET name redis EX 60
TTL name
```

再执行普通 `SET`：

```redis
SET name mysql
TTL name
```

此时通常返回：

```text
(integer) -1
```

因为普通 `SET` 覆盖 value 时，会清除原来的 TTL。

如果你希望更新 value 的同时保留 TTL，可以使用 `KEEPTTL`：

```redis
SET name postgres KEEPTTL
```

不过学习阶段最重要的是记住：更新缓存时要注意 TTL 是否被重置或清除。

---

## 九、业务场景：短信验证码

验证码天然适合 TTL。

设计 key：

```text
sms:code:phone:13800138000
```

写入：

```redis
SET sms:code:phone:13800138000 888888 EX 300
```

验证时读取：

```redis
GET sms:code:phone:13800138000
```

验证成功后删除：

```redis
DEL sms:code:phone:13800138000
```

为什么验证成功后还要删除？

因为验证码通常只能使用一次。即使它还没过期，也不应该继续可用。

---

## 十、业务场景：文章详情缓存

文章详情缓存可以设置 30 分钟过期：

```redis
SET article:detail:1001 '{"id":1001,"title":"Redis TTL"}' EX 1800
```

如果文章更新了，通常删除缓存：

```redis
DEL article:detail:1001
```

下次请求再从数据库读取新文章，并重新写入 Redis。

TTL 的作用是兜底：即使没有触发删除，缓存也会在一段时间后自动失效。

---

## 十一、TTL 应该设置多久

TTL 没有统一答案，要看业务。

| 场景 | 示例 TTL | 思路 |
| --- | --- | --- |
| 短信验证码 | 5 分钟 | 时间太长不安全，太短影响用户体验 |
| 登录 token | 几小时到几天 | 看安全策略和产品体验 |
| 文章详情缓存 | 10 到 60 分钟 | 内容变化频率不高，可以适当长一点 |
| 商品库存缓存 | 几秒到几十秒 | 一致性要求高，TTL 要短 |
| 空值缓存 | 30 秒到 5 分钟 | 防穿透，但不能长时间缓存不存在状态 |
| 限流计数 | 1 秒、1 分钟、1 小时 | 取决于限流窗口 |

TTL 规划要平衡：

- 数据新鲜度。
- 数据库压力。
- 用户体验。
- 内存占用。
- 业务安全性。

---

## 十二、TTL 随机化

如果大量 key 同一时间过期，可能导致缓存同时失效，大量请求打到数据库。

比如给 100 万个商品缓存都设置：

```text
EX 3600
```

如果它们几乎同一时间写入，就可能在一小时后集中失效。

常见做法是给 TTL 加随机值：

```text
基础 TTL 1800 秒 + 随机 0 到 600 秒
```

这样 key 会分散过期，降低缓存雪崩风险。

---

## 十三、常见错误

### 1. 忘记设置过期时间

验证码、限流计数、临时 token 忘记设置 TTL，会导致安全和内存问题。

### 2. 用两条命令写临时数据

```redis
SET code 888888
EXPIRE code 300
```

中间如果程序异常，可能留下永久 key。

更推荐：

```redis
SET code 888888 EX 300
```

### 3. 覆盖 key 后 TTL 消失

普通 `SET` 会清除旧 TTL。更新缓存时要特别注意。

### 4. TTL 设置太长或太短

太长会导致旧数据停留太久，太短会导致缓存命中率低。TTL 是业务设计，不是随手填数字。

---

## 十四、本节练习

请完成下面练习：

1. 写入 `login:code:1001`，值为 `888888`，过期时间 60 秒。
2. 使用 `TTL` 查看剩余时间。
3. 等几秒后再次查看 TTL，观察变化。
4. 使用 `PERSIST` 移除过期时间。
5. 再次使用 `TTL`，确认返回 `-1`。
6. 删除 `login:code:1001`。
7. 使用 `TTL` 查看不存在的 key，确认返回 `-2`。
8. 使用 `SET key value EX seconds` 写入一个临时 token。
9. 对一个带 TTL 的 key 执行普通 `SET`，观察 TTL 是否消失。
10. 思考：文章详情、短信验证码、商品库存缓存分别适合多长 TTL？

---

## 十五、本节小结

这一节你掌握了 Redis 过期时间：

- `EXPIRE key seconds`：给已有 key 设置过期时间。
- `TTL key`：查看 key 剩余秒数。
- `PERSIST key`：移除 key 的过期时间。
- `SET key value EX seconds`：写入时直接设置过期时间。
- TTL 设置在整个 key 上，不是某个字段上。
- 普通 `SET` 覆盖 key 时可能清除旧 TTL。
- TTL 规划要结合业务，不是随便写一个数字。

下一节我们会学习自增自减命令：`INCR`、`DECR`、`INCRBY`。它们是计数器、限流和阅读量统计的基础。
