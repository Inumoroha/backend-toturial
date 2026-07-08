# 1. String 字符串：缓存对象、验证码、计数器、锁基础

String 是 Redis 中最基础、最常用的数据结构。

前面你已经用过 `SET`、`GET`、`INCR`、`EXPIRE`，它们操作的主要就是 String。很多真实业务场景都可以先从 String 开始建模：缓存对象、验证码、登录 token、计数器、分布式锁基础等。

学完这一节后，你应该能够：

- 理解 Redis String 保存的是什么。
- 使用 String 缓存对象 JSON。
- 使用 String 保存验证码和登录 token。
- 使用 `INCR` 系列命令实现计数器。
- 理解 `SET NX EX` 为什么是分布式锁的基础。
- 知道 String 适合什么场景，不适合什么场景。

---

## 一、String 是什么

Redis String 可以理解为：一个 key 对应一个字符串 value。

示例：

```redis
SET name redis
GET name
```

返回：

```text
"redis"
```

这个 value 虽然叫字符串，但它可以保存很多内容：

- 普通文本。
- 数字。
- JSON 字符串。
- token。
- 验证码。
- 二进制数据。

比如：

```redis
SET count 10
SET login:token:user:1001 abcdefg
SET article:detail:1001 '{"id":1001,"title":"Redis String"}'
```

Redis 不关心 JSON 的内部结构。对 Redis 来说，它就是一段字符串。

---

## 二、常用命令总览

String 常用命令：

| 命令 | 作用 |
| --- | --- |
| `SET key value` | 设置字符串值 |
| `GET key` | 获取字符串值 |
| `MSET key value ...` | 批量设置 |
| `MGET key ...` | 批量获取 |
| `INCR key` | 整数自增 1 |
| `DECR key` | 整数自减 1 |
| `INCRBY key n` | 整数增加 n |
| `SET key value EX seconds` | 设置值并指定过期时间 |
| `SET key value NX EX seconds` | key 不存在时写入，并设置过期时间 |
| `GETSET key value` | 获取旧值并设置新值，了解即可 |

本阶段重点掌握前面几个命令即可。

---

## 三、场景一：缓存对象 JSON

最常见的缓存方式是把数据库对象序列化成 JSON，然后存到 Redis String。

假设数据库里有一篇文章：

```json
{
  "id": 1001,
  "title": "Redis String 入门",
  "author_id": 9
}
```

可以写入 Redis：

```redis
SET article:detail:1001 '{"id":1001,"title":"Redis String 入门","author_id":9}' EX 1800
```

读取：

```redis
GET article:detail:1001
```

业务流程通常是：

```text
请求文章详情
先查 Redis
命中：反序列化 JSON 并返回
未命中：查数据库，序列化 JSON，写入 Redis，再返回
```

这种方式简单直接，非常适合后端项目入门。

---

## 四、String 缓存对象的优点

String + JSON 的优点：

- 和后端对象序列化天然匹配。
- 读取时一次拿到完整对象。
- 写入和删除都简单。
- 适合 Cache-Aside 缓存模式。
- 便于跨语言系统读取，只要都能解析 JSON。

比如 Go 服务中常见流程：

```text
struct -> json.Marshal -> SET key json EX ttl
GET key -> json.Unmarshal -> struct
```

对于文章详情、商品详情、用户简要信息，String + JSON 是很常见的选择。

---

## 五、String 缓存对象的缺点

缺点也很明显：

- 不能直接更新 JSON 中某个字段。
- 每次读取都会拿到完整 JSON。
- value 太大时容易变成 Big Key。
- JSON 结构变化时要考虑兼容。
- 局部字段频繁变化时不够方便。

比如缓存用户资料：

```redis
SET user:profile:1001 '{"name":"tom","age":18,"city":"shanghai"}'
```

如果只想把 `city` 改成 `beijing`，Redis String 本身不会理解 JSON 字段。程序通常需要：

1. `GET user:profile:1001`。
2. 反序列化 JSON。
3. 修改 city。
4. 序列化 JSON。
5. `SET user:profile:1001 ...`。

如果你经常需要局部字段更新，Hash 可能更合适。

---

## 六、场景二：验证码

验证码是 String 的经典场景。

需求：手机号 `13800138000` 的验证码是 `888888`，5 分钟后过期。

key 设计：

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

这个场景体现了 String 的几个特点：

- value 很小。
- key 语义清晰。
- 有明确 TTL。
- 读写简单。

这是非常适合 Redis 的场景。

---

## 七、场景三：登录 token

登录 token 也适合用 String。

key 设计：

```text
login:token:user:1001
```

写入：

```redis
SET login:token:user:1001 token_value EX 604800
```

这里 `604800` 秒是 7 天。

接口鉴权时读取：

```redis
GET login:token:user:1001
```

退出登录时删除：

```redis
DEL login:token:user:1001
```

真实项目里还要考虑：

- 一个用户是否允许多端登录。
- token 是否需要续期。
- token 泄露后如何失效。
- Redis 故障时接口如何处理。

但从 Redis 数据结构角度看，String 足够表达这个模型。

---

## 八、场景四：计数器

String 可以保存整数，并配合 `INCR`、`DECR`、`INCRBY` 做计数。

文章阅读量：

```redis
INCR article:view_count:1001
```

点赞数：

```redis
INCR article:like_count:1001
```

批量增加：

```redis
INCRBY article:view_count:1001 10
```

读取：

```redis
GET article:view_count:1001
```

计数器是否设置 TTL 要看业务：

| 场景 | TTL 思路 |
| --- | --- |
| 文章总阅读量 | 通常长期保留，定时落库 |
| 每分钟访问次数 | 需要 TTL，窗口结束自动清理 |
| 每天签到次数 | 可以按天设置 TTL |
| 短信发送次数 | 需要 TTL，防止永久限制 |

---

## 九、场景五：分布式锁基础

Redis String 也常用于分布式锁的基础实现。

加锁的核心命令：

```redis
SET lock:order:1001 request_id NX EX 10
```

含义：

| 参数 | 含义 |
| --- | --- |
| `lock:order:1001` | 锁的 key |
| `request_id` | 锁的持有者标识 |
| `NX` | 只有 key 不存在时才写入 |
| `EX 10` | 10 秒后自动过期 |

如果返回 `OK`，表示抢锁成功。

如果返回 `(nil)`，表示锁已经存在，抢锁失败。

为什么要设置过期时间？

因为如果持有锁的服务崩溃，没有释放锁，过期时间可以避免锁永久不释放。

为什么 value 要放唯一标识？

因为释放锁时要确认锁还是自己持有的，避免误删别人的锁。

注意：分布式锁不是这一节的重点，后面会专门讲。现在你只需要知道：String 的 `SET NX EX` 是很多 Redis 锁实现的基础。

---

## 十、String 不适合什么场景

String 不适合：

- 需要频繁更新对象中的单个字段。
- 需要对集合成员做去重。
- 需要维护有序排行榜。
- 需要队列式从左进右出。
- 单个 value 特别大的对象。
- 需要按多个字段复杂查询。

比如用户标签：

```text
user:tags:1001 = "go,redis,mysql"
```

虽然能存，但不好判断某个标签是否存在，也不好去重。Set 更合适。

排行榜：

```text
rank:article = "1001:99,1002:88"
```

这样存很别扭，ZSet 更合适。

---

## 十一、常见错误

### 1. 把大对象塞进一个 String

比如把几万篇文章放进一个 JSON：

```text
article:all -> huge json
```

会导致读取、传输、删除都很重。

### 2. 忘记给临时数据设置 TTL

验证码、token、限流计数都应该规划 TTL。

### 3. 对非数字 String 执行 INCR

```redis
SET count hello
INCR count
```

会报错。

### 4. 用 String 硬做所有数据结构

String 很万能，但不是所有场景都应该用 String。Redis 的价值之一就是提供多种数据结构。

---

## 十二、本节练习

请完成下面练习：

1. 使用 String 缓存一篇文章详情 JSON，设置 30 分钟 TTL。
2. 使用 String 保存一个短信验证码，设置 5 分钟 TTL。
3. 使用 `GET` 读取验证码。
4. 验证成功后使用 `DEL` 删除验证码。
5. 使用 `INCR` 模拟文章阅读量增加 5 次。
6. 使用 `INCRBY` 一次增加 10 个阅读量。
7. 使用 `SET lock:order:1001 abc NX EX 10` 模拟抢锁。
8. 再执行一次同样命令，观察返回值。
9. 思考：用户资料适合用 String + JSON，还是 Hash？为什么？
10. 思考：排行榜为什么不适合用普通 String 表达？

---

## 十三、本节小结

String 是 Redis 最常用的数据结构。

你需要记住：

- String 可以保存文本、数字、JSON、token、验证码等。
- String + JSON 很适合缓存对象。
- String + TTL 很适合验证码、token、临时状态。
- String + `INCR` 很适合计数器。
- `SET key value NX EX seconds` 是分布式锁的基础。
- String 不适合需要集合、排序、队列、局部字段频繁更新的场景。

下一节我们学习 Hash，它更适合保存对象字段，例如用户资料和商品属性。
