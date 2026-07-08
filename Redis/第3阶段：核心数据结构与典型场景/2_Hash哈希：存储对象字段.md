# 2. Hash 哈希：存储对象字段

Hash 是 Redis 中非常适合保存对象字段的数据结构。

如果 String + JSON 更像“把整个对象打包成一段字符串”，那么 Hash 更像“把对象拆成多个字段保存”。例如用户资料、商品属性、文章统计信息，都可以考虑使用 Hash。

学完这一节后，你应该能够：

- 理解 Redis Hash 的结构。
- 使用 `HSET`、`HGET`、`HGETALL` 操作对象字段。
- 使用 `HMGET` 批量读取字段。
- 使用 `HINCRBY` 对字段做自增。
- 判断什么时候使用 Hash，什么时候使用 String + JSON。
- 知道 Hash 不适合什么场景。

---

## 一、Hash 是什么

Hash 可以理解为：一个 key 下面保存多个 field-value。

结构像这样：

```text
user:profile:1001
  name -> tom
  age  -> 18
  city -> shanghai
```

写入：

```redis
HSET user:profile:1001 name tom age 18 city shanghai
```

读取某个字段：

```redis
HGET user:profile:1001 name
```

返回：

```text
"tom"
```

读取所有字段：

```redis
HGETALL user:profile:1001
```

Hash 很适合表达“一个对象有多个字段”的场景。

---

## 二、常用命令总览

| 命令 | 作用 |
| --- | --- |
| `HSET key field value [field value ...]` | 设置一个或多个字段 |
| `HGET key field` | 获取单个字段 |
| `HMGET key field [field ...]` | 批量获取多个字段 |
| `HGETALL key` | 获取所有字段和值 |
| `HDEL key field [field ...]` | 删除字段 |
| `HEXISTS key field` | 判断字段是否存在 |
| `HLEN key` | 查看字段数量 |
| `HKEYS key` | 获取所有字段名 |
| `HVALS key` | 获取所有字段值 |
| `HINCRBY key field increment` | 字段整数自增 |

初学阶段重点掌握 `HSET`、`HGET`、`HMGET`、`HGETALL`、`HDEL`、`HINCRBY`。

---

## 三、HSET：设置字段

写入用户资料：

```redis
HSET user:profile:1001 name tom age 18 city shanghai
```

返回值表示新增了几个字段。

如果字段已经存在，再次 `HSET` 会更新字段值：

```redis
HSET user:profile:1001 city beijing
```

读取：

```redis
HGET user:profile:1001 city
```

返回：

```text
"beijing"
```

这就是 Hash 的优势：可以局部更新某个字段，而不用整体改写对象。

---

## 四、HGET 和 HMGET：读取字段

读取单个字段：

```redis
HGET user:profile:1001 name
```

读取多个字段：

```redis
HMGET user:profile:1001 name city
```

返回：

```text
1) "tom"
2) "beijing"
```

如果某个字段不存在，会返回 `(nil)`：

```redis
HMGET user:profile:1001 name birthday
```

可能返回：

```text
1) "tom"
2) (nil)
```

业务代码里要能处理字段缺失。

---

## 五、HGETALL：读取完整对象

读取全部字段：

```redis
HGETALL user:profile:1001
```

返回类似：

```text
1) "name"
2) "tom"
3) "age"
4) "18"
5) "city"
6) "beijing"
```

Redis 返回的是 field、value 交替排列的列表。

在客户端里通常会转换成 map：

```text
name -> tom
age -> 18
city -> beijing
```

注意：如果 Hash 字段特别多，`HGETALL` 也可能变成重操作。字段数量可控时才适合使用。

---

## 六、HDEL、HEXISTS、HLEN

删除字段：

```redis
HDEL user:profile:1001 city
```

判断字段是否存在：

```redis
HEXISTS user:profile:1001 city
```

返回：

```text
(integer) 0
```

查看字段数量：

```redis
HLEN user:profile:1001
```

这些命令适合做对象字段管理。

比如某个用户没有填写城市，就可以删除 `city` 字段，而不是删除整个用户资料 key。

---

## 七、HINCRBY：字段自增

Hash 的字段也可以做整数自增。

比如文章统计：

```redis
HSET article:stats:1001 view_count 0 like_count 0 comment_count 0
```

阅读量加 1：

```redis
HINCRBY article:stats:1001 view_count 1
```

点赞数加 1：

```redis
HINCRBY article:stats:1001 like_count 1
```

读取统计：

```redis
HGETALL article:stats:1001
```

Hash 很适合把同一个对象的多个计数字段放在一起。

---

## 八、TTL 仍然设置在整个 key 上

这是 Hash 最容易误解的地方。

你可以给整个 Hash 设置 TTL：

```redis
EXPIRE user:profile:1001 600
```

但不能给某个字段单独设置 TTL。

也就是说，下面这个对象会整体过期：

```text
user:profile:1001
  name
  age
  city
```

不能只让 `city` 字段 10 分钟后过期，而 `name` 字段不过期。

如果你确实需要字段级过期，通常要拆成多个 key，或者在字段值里保存业务过期时间。

---

## 九、场景一：用户资料

用户资料是 Hash 的典型场景。

key：

```text
user:profile:1001
```

写入：

```redis
HSET user:profile:1001 name tom age 18 city shanghai avatar /avatar/tom.png
```

读取昵称和头像：

```redis
HMGET user:profile:1001 name avatar
```

修改城市：

```redis
HSET user:profile:1001 city beijing
```

删除头像：

```redis
HDEL user:profile:1001 avatar
```

如果你的接口经常只读取或修改某几个字段，Hash 很自然。

---

## 十、场景二：商品属性

商品属性也适合 Hash。

```redis
HSET product:attr:2001 name keyboard brand keychron layout 87 switch red
```

读取商品品牌和轴体：

```redis
HMGET product:attr:2001 brand switch
```

修改库存不一定放在这个 Hash 里。库存可能变化频繁，且一致性要求更高，通常要单独设计。

这就是 key 设计要结合业务变化频率的原因。

---

## 十一、场景三：文章统计信息

文章统计可以放在一个 Hash：

```redis
HSET article:stats:1001 view_count 0 like_count 0 comment_count 0
```

增加阅读量：

```redis
HINCRBY article:stats:1001 view_count 1
```

增加评论数：

```redis
HINCRBY article:stats:1001 comment_count 1
```

读取全部统计：

```redis
HGETALL article:stats:1001
```

这种设计比给每个字段单独建 key 更集中：

```text
article:stats:1001
  view_count
  like_count
  comment_count
```

但如果某个字段访问极其频繁，也可以单独拆 key。没有绝对答案，要看访问模式。

---

## 十二、Hash 与 String + JSON 怎么选

这是很常见的问题。

| 对比项 | String + JSON | Hash |
| --- | --- | --- |
| 保存方式 | 整个对象序列化成字符串 | 对象字段拆开保存 |
| 读取完整对象 | 很方便 | 需要 `HGETALL` 或多个字段 |
| 局部字段更新 | 不方便 | 很方便 |
| 嵌套结构 | 更自然 | 不适合复杂嵌套 |
| 与后端对象转换 | 很自然 | 需要 map 和字段转换 |
| 字段级操作 | 不支持 | 支持 |

适合 String + JSON：

- 每次都读取完整对象。
- 对象结构有嵌套。
- 后端直接序列化和反序列化。
- 更新时通常整体删除缓存。

适合 Hash：

- 经常读写单个字段。
- 对象字段比较扁平。
- 多个计数字段放在同一个对象下。
- 希望避免每次整体 JSON 反序列化。

---

## 十三、Hash 不适合什么场景

Hash 不适合：

- 字段数量无限增长。
- 需要给字段单独设置 TTL。
- value 是复杂深层嵌套 JSON。
- 需要按字段值做复杂查询。
- 需要对字段排序、排名。

比如用户所有订单不适合塞进一个 Hash：

```text
user:orders:1001
  order_1 -> ...
  order_2 -> ...
  order_999999 -> ...
```

这会让 Hash 变得越来越大，也不利于分页和查询。

Redis 不是关系型数据库，不要把它当成可以任意查询的表。

---

## 十四、常见错误

### 1. 对 Hash 使用 GET

```redis
GET user:profile:1001
```

会报 `WRONGTYPE`。

应该使用：

```redis
HGET user:profile:1001 name
```

### 2. 滥用 HGETALL

字段很多时，`HGETALL` 会一次返回全部内容，可能变慢。

### 3. 以为字段可以单独 TTL

Hash 的 TTL 设置在整个 key 上。

### 4. 把无限增长数据塞进 Hash

比如一个用户所有行为日志、所有订单，不适合放在一个 Hash 中。

---

## 十五、本节练习

请完成下面练习：

1. 使用 Hash 保存 `user:profile:1001`，包含 `name`、`age`、`city`。
2. 使用 `HGET` 读取 `name`。
3. 使用 `HMGET` 读取 `name` 和 `city`。
4. 使用 `HSET` 修改 `city`。
5. 使用 `HGETALL` 查看完整用户资料。
6. 使用 `HDEL` 删除 `city`。
7. 使用 `HEXISTS` 判断 `city` 是否存在。
8. 使用 Hash 保存文章统计，并用 `HINCRBY` 增加阅读量。
9. 给 Hash 设置 TTL，并观察 TTL 是设置在整个 key 上的。
10. 思考：用户资料、文章详情、商品属性分别适合 Hash 还是 String + JSON？

---

## 十六、本节小结

Hash 适合保存对象字段。

你需要记住：

- Hash 是一个 key 下多个 field-value。
- `HSET` 可以设置一个或多个字段。
- `HGET` 读取单字段，`HMGET` 读取多个字段，`HGETALL` 读取全部字段。
- `HINCRBY` 可以对数字字段自增。
- Hash 适合扁平对象和局部字段更新。
- Hash 不适合复杂嵌套、字段无限增长、字段单独 TTL 的场景。

下一节我们学习 List，它适合简单队列、最新列表和任务列表。
