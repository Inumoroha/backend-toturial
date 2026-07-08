# 3. 理解 Redis 的 key-value 模型

前两节我们已经启动 Redis，并用命令行和图形化客户端连接了它。

这一节开始建立 Redis 的第一层直觉：Redis 是一个以 key-value 为核心的数据系统。你写入数据时，先设计 key，再选择 value 的结构。

学完这一节后，你应该能够：

- 理解 Redis 中 key 和 value 分别是什么。
- 知道 Redis 不是只能保存字符串。
- 理解 key 命名为什么重要。
- 知道 TTL 与 key-value 模型的关系。
- 初步判断一个业务数据适不适合放进 Redis。

---

## 一、什么是 key-value

key-value 可以理解为：用一个名字找到一份数据。

比如：

```redis
SET user:1:name tom
GET user:1:name
```

这里：

| 部分 | 含义 |
| --- | --- |
| `user:1:name` | key，用来定位数据 |
| `tom` | value，真正保存的数据 |

Redis 中很多操作都是围绕 key 展开的：

```redis
EXISTS user:1:name
TYPE user:1:name
TTL user:1:name
DEL user:1:name
```

你可以先记住一句话：在 Redis 里，key 是入口，value 是内容。

---

## 二、Redis 的 key 是字符串

Redis 的 key 本质上是字符串。

下面这些都可以是 key：

```text
name
user:1:name
article:1001:detail
login:token:abc123
rate_limit:ip:127.0.0.1
```

虽然 Redis 不强制你用什么命名规则，但实际项目里一定要有规范。

不推荐：

```text
a
user1
temp
abc
```

推荐：

```text
user:profile:1001
article:detail:9527
login:token:user:1001
sms:code:phone:13800138000
```

好的 key 命名能让你一眼看出它属于什么业务、保存什么数据、和哪个对象有关。

---

## 三、冒号不是目录，只是命名约定

你经常会看到这种 key：

```text
user:profile:1001
```

冒号 `:` 不是 Redis 的目录语法。Redis 不会真的创建 `user` 文件夹，也不会创建 `profile` 子目录。

这只是一种命名约定。

它的好处是：

- 方便阅读。
- 方便按业务分类。
- 方便图形化客户端展示层级。
- 方便使用 `SCAN` 按模式查找。

比如：

```redis
SCAN 0 MATCH user:profile:* COUNT 100
```

这表示分批查找符合 `user:profile:*` 模式的 key。

注意：学习阶段可以知道这个命令，但不要急着深入。后面讲 key 设计时会专门讲 `SCAN` 和为什么生产环境慎用 `KEYS *`。

---

## 四、Redis 的 value 不只有字符串

虽然我们现在主要使用：

```redis
SET key value
```

但 Redis 的 value 可以是多种数据结构。

常见类型包括：

| 类型 | 适合保存什么 |
| --- | --- |
| String | 字符串、数字、JSON、验证码、缓存内容 |
| Hash | 对象字段，比如用户资料、商品属性 |
| List | 列表、队列、最新动态 |
| Set | 去重集合、标签、抽奖用户 |
| ZSet | 排行榜、带分数排序的数据 |

也就是说，Redis 不是简单的字符串仓库。它更像是一个支持多种内存数据结构的 key-value 系统。

当前阶段你只需要先理解：每个 value 都挂在某个 key 下面。

---

## 五、用 String 保存对象

假设你要缓存一篇文章详情，可以把文章序列化成 JSON 字符串：

```redis
SET article:detail:1001 '{"id":1001,"title":"Redis 入门","author":"tom"}' EX 1800
```

读取：

```redis
GET article:detail:1001
```

这种方式很常见，尤其是在后端项目里：

1. 从数据库查出文章。
2. 序列化成 JSON。
3. 写入 Redis。
4. 下次请求直接从 Redis 读取 JSON。

优点是简单。

缺点是如果只想改文章标题，通常需要把整个 JSON 读出来、反序列化、修改、再整体写回去。

---

## 六、用 Hash 保存对象字段

同样是用户资料，也可以使用 Hash：

```redis
HSET user:profile:1001 name tom age 18 city shanghai
```

读取某个字段：

```redis
HGET user:profile:1001 name
```

读取所有字段：

```redis
HGETALL user:profile:1001
```

Hash 的特点是可以按字段读写。

比如只修改城市：

```redis
HSET user:profile:1001 city beijing
```

String 存 JSON 和 Hash 存字段都可以保存对象，但取舍不同：

| 方式 | 优点 | 缺点 |
| --- | --- | --- |
| String + JSON | 简单，和后端对象序列化配合自然 | 局部字段更新不方便 |
| Hash | 可以单独读写字段 | 对复杂嵌套结构不够自然 |

初学阶段不用纠结哪个绝对更好。真实项目里要看读取方式、更新方式、数据大小和团队习惯。

---

## 七、TTL 属于 key

Redis 的过期时间是设置在 key 上的。

比如：

```redis
SET login:code:1001 888888 EX 300
```

表示整个 `login:code:1001` 这个 key 会在 300 秒后过期。

查看 TTL：

```redis
TTL login:code:1001
```

删除过期时间：

```redis
PERSIST login:code:1001
```

需要注意：TTL 不是设置在某个字段上的，而是设置在整个 key 上。

例如 Hash：

```redis
HSET user:profile:1001 name tom age 18
EXPIRE user:profile:1001 600
```

这里过期的是整个 `user:profile:1001`，不是单独的 `name` 字段或 `age` 字段。

---

## 八、什么数据适合放 Redis

Redis 适合保存：

- 热点数据缓存。
- 验证码。
- 登录 token。
- 临时状态。
- 计数器。
- 限流数据。
- 排行榜。
- 去重集合。
- 短时间内高频访问的数据。

比如：

```text
article:detail:1001
sms:code:phone:13800138000
login:token:user:1001
article:view_count:1001
rank:article:daily
```

这些数据要么访问频繁，要么有明显过期时间，要么适合用 Redis 的数据结构表达。

---

## 九、什么数据不适合直接放 Redis

Redis 不适合无脑保存所有业务数据。

不建议把这些数据只放 Redis：

- 订单主数据。
- 支付流水。
- 用户核心资料唯一副本。
- 审计日志唯一副本。
- 必须长期可靠保存的数据。

原因很简单：Redis 通常以内存为主，虽然支持持久化，但它更常见的定位是缓存和高性能数据结构服务，不是关系型数据库的完整替代品。

一个常见搭配是：

```text
MySQL 或 PostgreSQL 保存核心数据
Redis 保存缓存、计数、限流、临时状态
```

---

## 十、用 key-value 思维设计一个验证码

短信验证码是很适合 Redis 的例子。

业务需求：

- 手机号 `13800138000` 收到验证码 `888888`。
- 验证码 5 分钟后过期。
- 用户提交验证码时读取 Redis 判断是否正确。

key 可以设计为：

```text
sms:code:phone:13800138000
```

写入：

```redis
SET sms:code:phone:13800138000 888888 EX 300
```

读取：

```redis
GET sms:code:phone:13800138000
```

验证成功后删除：

```redis
DEL sms:code:phone:13800138000
```

这个例子体现了 Redis 的几个特点：

- key 能表达业务含义。
- value 保存实际验证码。
- TTL 自动处理过期。
- 读取和删除都很快。

---

## 十一、本节练习

请用 `redis-cli` 完成下面练习：

1. 写入 `user:profile:1001`，value 是一段 JSON 字符串。
2. 使用 `GET` 读取它。
3. 使用 `TYPE` 查看它的数据类型。
4. 写入 `sms:code:phone:13800138000`，验证码是 `888888`，过期时间 300 秒。
5. 使用 `TTL` 查看剩余时间。
6. 使用 Hash 写入 `user:profile:1002`，包含 `name`、`age`、`city`。
7. 使用 `HGET` 读取 `name` 字段。
8. 使用 `HGETALL` 读取完整用户资料。
9. 给 `user:profile:1002` 设置 600 秒过期时间。
10. 思考：如果是文章详情缓存，你更倾向用 String + JSON，还是 Hash？为什么？

---

## 十二、本节小结

这一节你理解了 Redis 的 key-value 模型。

你需要记住：

- Redis 通过 key 找到 value。
- key 本质上是字符串，但应该有清晰命名规范。
- 冒号只是命名约定，不是真实目录。
- value 可以是 String、Hash、List、Set、ZSet 等结构。
- TTL 设置在整个 key 上。
- Redis 适合缓存、临时状态、计数、限流、排行榜等场景。
- Redis 不应该无脑替代 MySQL 或 PostgreSQL 保存所有核心业务数据。

下一节我们会继续理解 Redis 为什么快：单线程主循环和内存读写。
