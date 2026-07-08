# 1. 基础命令：SET、GET、DEL、EXISTS、TYPE

第二阶段的目标是熟练管理 Redis key，并设计清晰、可维护、不容易冲突的 key 命名方式。

这一节先从最常用的基础命令开始。它们看起来简单，但会贯穿后面所有 Redis 使用场景。

学完这一节后，你应该能够：

- 使用 `SET` 写入 String 类型数据。
- 使用 `GET` 读取 String 类型数据。
- 使用 `DEL` 删除 key。
- 使用 `EXISTS` 判断 key 是否存在。
- 使用 `TYPE` 查看 key 对应 value 的数据类型。
- 理解这些命令在缓存、验证码、临时状态中的基本用法。

---

## 一、准备 Redis 环境

如果 Redis 容器还没有启动，可以进入第一阶段目录执行：

```bash
docker compose up -d
```

也可以直接用 Docker 启动：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine
```

进入 `redis-cli`：

```bash
docker exec -it redis-dev redis-cli
```

先测试连接：

```redis
PING
```

返回 `PONG` 说明连接正常。

---

## 二、SET：写入一个 key

`SET` 用来设置一个 String 类型的 key-value。

语法：

```redis
SET key value
```

示例：

```redis
SET name redis
```

表示写入：

```text
key: name
value: redis
```

读取：

```redis
GET name
```

返回：

```text
"redis"
```

Redis 中的 String 不只是普通文本，也可以保存数字、JSON 字符串、token、验证码等。

例如：

```redis
SET user:1:name tom
SET article:1001:title "Redis 入门教程"
SET login:token:1001 abcdefg
SET article:1001:detail '{"id":1001,"title":"Redis 入门教程"}'
```

---

## 三、SET 会覆盖旧值

如果 key 已经存在，再执行 `SET` 会覆盖旧值。

```redis
SET name redis
GET name
SET name mysql
GET name
```

最后返回：

```text
"mysql"
```

这点很重要。缓存更新、token 刷新、验证码重发时，覆盖旧值是常见行为。

但如果你不希望覆盖已有 key，可以使用 `NX` 参数：

```redis
SET name postgres NX
```

`NX` 表示只有 key 不存在时才写入。

如果你希望只有 key 已存在时才更新，可以用 `XX`：

```redis
SET name redis XX
```

这两个参数后面学习分布式锁和缓存更新时还会遇到。

---

## 四、GET：读取 String 类型 key

`GET` 用来读取 String 类型 key 的 value。

语法：

```redis
GET key
```

示例：

```redis
SET user:1:name tom
GET user:1:name
```

返回：

```text
"tom"
```

如果 key 不存在：

```redis
GET not_exists_key
```

返回：

```text
(nil)
```

`(nil)` 表示 Redis 中没有这个 key。

在 Go 项目里，使用 `go-redis` 时，读取不存在的 key 通常会得到 `redis.Nil`，这和真正的系统错误要区分开。

---

## 五、GET 只能读取 String

如果 key 对应的 value 不是 String，使用 `GET` 会报错。

比如先写入一个 Hash：

```redis
HSET user:profile:1 name tom age 18
```

再执行：

```redis
GET user:profile:1
```

会得到类似错误：

```text
WRONGTYPE Operation against a key holding the wrong kind of value
```

这说明 `user:profile:1` 存在，但它不是 String 类型。

遇到这种情况，先用 `TYPE` 查看类型。

---

## 六、TYPE：查看 key 的数据类型

`TYPE` 用来查看 key 对应 value 的类型。

语法：

```redis
TYPE key
```

示例：

```redis
SET user:1:name tom
TYPE user:1:name
```

返回：

```text
string
```

写入一个 Hash：

```redis
HSET user:profile:1 name tom age 18
TYPE user:profile:1
```

返回：

```text
hash
```

如果 key 不存在：

```redis
TYPE not_exists_key
```

返回：

```text
none
```

`TYPE` 在排查问题时非常有用。比如程序报 `WRONGTYPE`，第一步就应该查 `TYPE`。

---

## 七、EXISTS：判断 key 是否存在

`EXISTS` 用来判断 key 是否存在。

语法：

```redis
EXISTS key [key ...]
```

示例：

```redis
SET name redis
EXISTS name
```

返回：

```text
(integer) 1
```

删除后再查：

```redis
DEL name
EXISTS name
```

返回：

```text
(integer) 0
```

`EXISTS` 也可以一次判断多个 key：

```redis
SET a 1
SET b 2
EXISTS a b c
```

如果 `a` 和 `b` 存在，`c` 不存在，返回：

```text
(integer) 2
```

返回值表示存在的 key 数量，不是简单的 true 或 false。

---

## 八、DEL：删除 key

`DEL` 用来删除一个或多个 key。

语法：

```redis
DEL key [key ...]
```

示例：

```redis
SET name redis
DEL name
```

返回：

```text
(integer) 1
```

表示删除了 1 个 key。

如果删除不存在的 key：

```redis
DEL not_exists_key
```

返回：

```text
(integer) 0
```

一次删除多个 key：

```redis
SET a 1
SET b 2
SET c 3
DEL a b c
```

返回：

```text
(integer) 3
```

注意：删除大 key 时，`DEL` 可能产生阻塞。后面学习生产排查时，会讲更适合删除大 key 的 `UNLINK`。

当前阶段先记住：普通小 key 可以直接 `DEL`。

---

## 九、业务场景：文章详情缓存

假设一个 Go 后端接口要查询文章详情。

可以设计 key：

```text
article:detail:1001
```

第一次请求时，数据库查出文章后写入 Redis：

```redis
SET article:detail:1001 '{"id":1001,"title":"Redis 基础命令","content":"..."}'
```

后续请求优先读取 Redis：

```redis
GET article:detail:1001
```

更新文章后删除缓存：

```redis
DEL article:detail:1001
```

为什么更新后通常是删除缓存，而不是直接更新缓存？

因为删除更简单，更不容易因为字段遗漏、序列化差异、并发写入导致缓存和数据库不一致。下次读取时再从数据库加载最新数据并写回 Redis。

---

## 十、常见错误

### 1. key 拼错

写入：

```redis
SET user:profile:1001 tom
```

读取时写成：

```redis
GET user:profiles:1001
```

会返回 `(nil)`。

这类问题在项目里很常见，所以 key 命名要集中封装，不要在代码各处手写字符串。

### 2. 类型用错

对 Hash 使用 `GET`，对 String 使用 `HGET`，都会报 `WRONGTYPE`。

排查方式：

```redis
TYPE your:key
```

### 3. 误删 key

`DEL` 很直接，删了就是删了。

学习环境没关系，生产环境要谨慎，尤其不要批量删除没有确认过的 key。

---

## 十一、本节练习

请在 `redis-cli` 中完成：

1. 写入 `user:1:name`，值为 `tom`。
2. 使用 `GET` 读取 `user:1:name`。
3. 使用 `EXISTS` 判断 `user:1:name` 是否存在。
4. 使用 `TYPE` 查看 `user:1:name` 的类型。
5. 再次 `SET user:1:name jack`，观察旧值是否被覆盖。
6. 写入 `article:detail:1001`，value 是一段 JSON 字符串。
7. 使用 `GET` 读取文章详情。
8. 使用 `DEL` 删除文章详情。
9. 删除后使用 `GET` 和 `EXISTS` 分别观察结果。
10. 创建一个 Hash，再尝试用 `GET` 读取它，观察 `WRONGTYPE` 错误。

---

## 十二、本节小结

这一节你掌握了 Redis 最基础的一组命令：

- `SET`：写入 String。
- `GET`：读取 String。
- `DEL`：删除 key。
- `EXISTS`：判断 key 是否存在，返回存在的 key 数量。
- `TYPE`：查看 key 对应 value 的类型。

这几个命令是 Redis 学习的地基。后面学习缓存、验证码、限流、排行榜、分布式锁时，你都会反复用到它们。
