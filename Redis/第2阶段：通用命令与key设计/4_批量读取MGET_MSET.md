# 4. 批量读取：MGET、MSET

在后端项目里，一次请求经常需要读取多个缓存。如果每个 key 都单独发一次 Redis 请求，网络往返会变多，性能也会变差。

这一节学习 `MGET` 和 `MSET`，它们用于批量读取和批量写入 String 类型 key。

学完这一节后，你应该能够：

- 使用 `MSET` 一次写入多个 key。
- 使用 `MGET` 一次读取多个 key。
- 理解批量操作如何减少网络往返。
- 知道 `MGET` 遇到不存在 key 时的返回结果。
- 理解批量缓存读取时如何处理部分命中。

---

## 一、为什么需要批量操作

假设页面要展示 3 篇文章标题。

如果你这样读取：

```redis
GET article:title:1001
GET article:title:1002
GET article:title:1003
```

客户端需要向 Redis 发送 3 次请求，Redis 返回 3 次结果。

如果使用 `MGET`：

```redis
MGET article:title:1001 article:title:1002 article:title:1003
```

只需要一次请求，就能拿到多个结果。

Redis 自身处理 `GET` 很快，但网络往返不是免费的。批量读取可以明显减少 RTT 开销。

---

## 二、MSET：批量写入

`MSET` 用来一次设置多个 String key。

语法：

```redis
MSET key value [key value ...]
```

示例：

```redis
MSET article:title:1001 "Redis 入门" article:title:1002 "缓存设计" article:title:1003 "Go 接入 Redis"
```

读取其中一个：

```redis
GET article:title:1001
```

返回：

```text
"Redis 入门"
```

`MSET` 会覆盖已有 key。

比如：

```redis
MSET article:title:1001 "新的标题"
GET article:title:1001
```

返回：

```text
"新的标题"
```

---

## 三、MGET：批量读取

`MGET` 用来一次读取多个 String key。

语法：

```redis
MGET key [key ...]
```

示例：

```redis
MGET article:title:1001 article:title:1002 article:title:1003
```

返回：

```text
1) "Redis 入门"
2) "缓存设计"
3) "Go 接入 Redis"
```

返回结果的顺序和请求 key 的顺序一致。

这一点很重要。后端代码里通常要按传入 key 的顺序把结果映射回业务 ID。

---

## 四、不存在的 key 会返回 nil

如果某些 key 不存在：

```redis
DEL article:title:1002
MGET article:title:1001 article:title:1002 article:title:1003
```

返回可能是：

```text
1) "Redis 入门"
2) (nil)
3) "Go 接入 Redis"
```

这叫部分命中。

业务代码需要识别：

- 哪些 key 命中了 Redis。
- 哪些 key 没命中，需要回源数据库。
- 回源后是否写回 Redis。

---

## 五、业务场景：批量文章标题缓存

假设接口要展示文章列表，每页 10 篇文章。

可以设计 key：

```text
article:title:1001
article:title:1002
article:title:1003
```

批量读取：

```redis
MGET article:title:1001 article:title:1002 article:title:1003
```

如果全部命中，直接返回。

如果部分未命中：

```text
1001 命中
1002 未命中
1003 命中
```

程序可以只查数据库中的 `1002`，查到后写回 Redis。

这比每篇文章都查数据库更高效。

---

## 六、业务场景：用户简要信息

社交应用里，经常要批量展示用户头像和昵称。

key 可以设计为：

```text
user:summary:1001
user:summary:1002
user:summary:1003
```

value 保存 JSON：

```json
{"id":1001,"nickname":"tom","avatar":"/a.png"}
```

批量写入测试数据：

```redis
MSET user:summary:1001 '{"id":1001,"nickname":"tom"}' user:summary:1002 '{"id":1002,"nickname":"jack"}'
```

批量读取：

```redis
MGET user:summary:1001 user:summary:1002 user:summary:1003
```

如果 `user:summary:1003` 返回 `(nil)`，就说明这个用户简要信息没有缓存。

---

## 七、MSET 不能直接给每个 key 设置不同 TTL

`MSET` 只能批量设置 key-value，不能像 `SET key value EX seconds` 那样给每个 key 同时设置 TTL。

比如下面这种写法是不对的：

```redis
MSET a 1 EX 60 b 2 EX 60
```

Redis 会把 `EX` 和 `60` 当成普通 key-value 的一部分，导致结果不是你想要的。

如果批量写入还要设置 TTL，常见选择有：

1. 多次执行 `SET key value EX seconds`。
2. 使用 Pipeline 批量发送多条命令。
3. 使用 Lua 脚本封装。

当前阶段先记住：`MSET` 适合批量写入 String，但 TTL 要额外设计。

---

## 八、MGET 只能读取 String 类型

和 `GET` 一样，`MGET` 面向 String 类型 key。

如果某个 key 是 Hash、List、Set 等类型，`MGET` 会报类型错误。

排查方式：

```redis
TYPE your:key
```

所以批量读取前，要保证这些 key 的数据结构一致。

同一个 key 前缀下，最好使用同一种 value 类型。

比如：

```text
user:summary:1001 -> String JSON
user:summary:1002 -> String JSON
user:summary:1003 -> String JSON
```

不要一会儿存 String，一会儿存 Hash。

---

## 九、批量不是越大越好

`MGET` 可以减少网络往返，但一次读取太多 key 也会带来问题：

- Redis 单次命令处理时间变长。
- 返回结果太大，占用网络带宽。
- 客户端反序列化压力变大。
- 某个请求可能拖慢其他请求。

比如一次 `MGET` 10 个 key 通常很正常。

一次 `MGET` 10 万个 key 就很危险。

真实项目里要控制批量大小，例如每批几十个或几百个，具体看 value 大小和接口延迟要求。

---

## 十、MGET 与 Pipeline 的区别

`MGET` 是一个 Redis 命令，一次读取多个 String key。

Pipeline 是客户端能力，可以一次发送多条 Redis 命令，减少网络往返。

比如：

```text
MGET: 一条命令，多个 key
Pipeline: 多条命令，一起发出去
```

如果都是 String key，`MGET` 很方便。

如果你要批量执行不同命令，比如：

```redis
GET article:detail:1001
TTL article:detail:1001
GET article:detail:1002
TTL article:detail:1002
```

就更适合 Pipeline。

Pipeline 后面在 Go 接入 Redis 时会重点讲。

---

## 十一、常见错误

### 1. 忘记处理 nil

`MGET` 返回列表中可能夹着 `(nil)`。代码里不能假设全部命中。

### 2. 返回顺序和业务 ID 对不上

`MGET` 返回结果按 key 顺序排列。后端代码要维护 key 与 ID 的映射。

### 3. 一次批量太大

批量可以减少网络开销，但过大的批量会制造新的慢请求。

### 4. 以为 MSET 可以设置 TTL

`MSET` 不能给每个 key 设置 TTL。要用其他方式处理。

---

## 十二、本节练习

请完成下面练习：

1. 使用 `MSET` 写入 3 个文章标题。
2. 使用 `MGET` 一次读取这 3 个标题。
3. 删除其中一个 key。
4. 再次使用 `MGET`，观察 `(nil)` 的位置。
5. 写入 3 个用户简要信息 JSON。
6. 使用 `MGET` 批量读取用户信息。
7. 思考：如果 10 个 key 中只有 6 个命中，程序应该怎么查数据库？
8. 尝试对一个 Hash key 使用 `MGET`，观察错误。
9. 思考：为什么一次 `MGET` 不应该读取特别多 key？
10. 思考：如果批量写入还要 TTL，你会怎么做？

---

## 十三、本节小结

这一节你掌握了批量读取和写入：

- `MSET`：一次写入多个 String key。
- `MGET`：一次读取多个 String key。
- `MGET` 返回结果顺序和请求 key 顺序一致。
- 不存在的 key 在结果中返回 `(nil)`。
- `MSET` 不能直接给每个 key 设置 TTL。
- 批量操作能减少网络往返，但批量大小要控制。

下一节我们会学习安全遍历：`SCAN`。它是线上查找 key 时比 `KEYS *` 更合适的方式。
