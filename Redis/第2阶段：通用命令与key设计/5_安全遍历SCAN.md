# 5. 安全遍历：SCAN

查找 key 是 Redis 排查中很常见的需求。比如你想知道某类缓存是否写入、某个用户相关 key 有哪些、某个前缀下有多少数据。

很多初学者第一反应是使用 `KEYS *`。但在生产环境中，`KEYS *` 很危险。

这一节学习更安全的遍历方式：`SCAN`。

学完这一节后，你应该能够：

- 理解为什么生产环境慎用 `KEYS *`。
- 使用 `SCAN` 分批遍历 key。
- 使用 `MATCH` 按模式查找 key。
- 使用 `COUNT` 给 Redis 一个扫描数量提示。
- 理解 `SCAN` 的游标机制。
- 知道 `SCAN` 不是精确分页，也不是强一致快照。

---

## 一、为什么不能随便用 KEYS *

`KEYS` 可以按模式查找 key。

比如：

```redis
KEYS *
```

表示查找所有 key。

在本地学习环境中，只有几十个 key，执行它看起来没问题。

但生产环境可能有几十万、几百万甚至更多 key。`KEYS *` 会一次性扫描整个 key 空间，并且在执行期间可能阻塞其他命令。

Redis 核心命令处理是单线程主循环。如果一个命令执行时间太长，其他请求就要排队等待。

所以线上排查时，不要把 `KEYS *` 当成默认工具。

---

## 二、SCAN 的基本思想

`SCAN` 是渐进式遍历。

它不会一次把所有 key 都扫完，而是每次返回一部分 key，并给你一个新的游标。

你拿着游标继续扫，直到游标回到 `0`，表示本轮遍历结束。

基本语法：

```redis
SCAN cursor
```

第一次从 `0` 开始：

```redis
SCAN 0
```

返回类似：

```text
1) "12"
2) 1) "user:1:name"
   2) "article:detail:1001"
   3) "login:token:1001"
```

第一部分 `"12"` 是下次要使用的游标。

第二部分是本次返回的 key 列表。

继续：

```redis
SCAN 12
```

直到某次返回的游标是 `0`。

---

## 三、准备一些测试 key

先写入一批 key：

```redis
MSET user:profile:1001 tom user:profile:1002 jack user:profile:1003 lucy
MSET article:detail:1001 a article:detail:1002 b article:detail:1003 c
MSET login:token:1001 t1 login:token:1002 t2
```

然后执行：

```redis
SCAN 0
```

你会看到 Redis 返回一部分 key 和一个新游标。

注意：本地 key 很少时，`SCAN 0` 可能一次就返回所有 key，并且游标直接回到 `0`。这很正常。

---

## 四、MATCH：按模式查找

`MATCH` 用来筛选 key。

语法：

```redis
SCAN cursor MATCH pattern
```

查找用户资料 key：

```redis
SCAN 0 MATCH user:profile:*
```

查找文章详情 key：

```redis
SCAN 0 MATCH article:detail:*
```

查找登录 token：

```redis
SCAN 0 MATCH login:token:*
```

常见模式：

| 模式 | 含义 |
| --- | --- |
| `user:*` | 以 `user:` 开头 |
| `article:detail:*` | 文章详情缓存 |
| `*:1001` | 以 `:1001` 结尾 |
| `*token*` | 包含 `token` |

实际项目中，清晰的 key 命名能让 `MATCH` 更好用。

---

## 五、COUNT：控制每次扫描数量提示

`COUNT` 用来告诉 Redis 每次大概扫描多少元素。

语法：

```redis
SCAN cursor COUNT count
```

示例：

```redis
SCAN 0 COUNT 100
```

结合 `MATCH`：

```redis
SCAN 0 MATCH user:profile:* COUNT 100
```

注意：`COUNT` 只是提示，不是精确数量。

你写 `COUNT 100`，Redis 不保证每次一定返回 100 个 key。它只是倾向于多扫描一些。

所以不要把 `SCAN` 当成传统数据库的分页查询。

---

## 六、完整遍历流程

完整遍历的逻辑是：

```text
cursor = 0
循环：
  执行 SCAN cursor MATCH pattern COUNT 100
  处理返回的 key
  cursor = 返回的新 cursor
直到 cursor == 0
```

在命令行里，你需要手动复制游标继续执行。

在 Go 代码里，客户端通常会提供迭代器，帮你处理游标。

例如伪代码：

```text
cursor := 0
for {
    keys, nextCursor := scan(cursor)
    handle(keys)
    if nextCursor == 0 {
        break
    }
    cursor = nextCursor
}
```

---

## 七、SCAN 可能返回重复 key

`SCAN` 在遍历期间，Redis 中的 key 可能正在新增、删除、过期。

因此它不是一个强一致快照。

这意味着：

- 可能返回重复 key。
- 可能漏掉遍历过程中新增的 key。
- 返回顺序不固定。
- 不能用它做严格分页。

所以使用 `SCAN` 时，业务逻辑要能接受这些特性。

如果你只是线上排查、渐进式清理、统计某类 key，`SCAN` 很合适。

如果你需要强一致列表，Redis key 空间扫描不是合适方案。

---

## 八、业务场景：查找某类缓存

比如你想查看文章详情缓存：

```redis
SCAN 0 MATCH article:detail:* COUNT 100
```

这能帮助你确认：

- 缓存有没有写入。
- key 命名是否符合预期。
- 某类 key 数量是否异常。

如果你看到奇怪的 key：

```text
article_details_1001
article:detail:1001
article:details:1002
```

说明项目里 key 命名可能不统一。

这就是为什么 key 设计要提前规范。

---

## 九、业务场景：清理某个前缀的测试 key

学习环境中，你可以用 `SCAN` 查找测试 key：

```redis
SCAN 0 MATCH debug:* COUNT 100
```

然后逐个删除：

```redis
DEL debug:message:1 debug:message:2
```

生产环境中，批量删除要更谨慎，尤其 value 可能很大时。

后面学习生产排查会讲 `UNLINK` 和分批删除策略。

---

## 十、SCAN 和 KEYS 的区别

| 维度 | KEYS | SCAN |
| --- | --- | --- |
| 扫描方式 | 一次性扫描 | 分批渐进扫描 |
| 对 Redis 影响 | key 很多时可能阻塞明显 | 更温和 |
| 返回结果 | 一次返回全部匹配 key | 每次返回一部分 |
| 使用场景 | 本地、小数据量调试 | 线上排查、分批处理 |
| 是否适合生产 | 慎用 | 更推荐 |

这不是说 `KEYS` 永远不能用。

在本地学习、测试库、确认 key 很少的环境中，`KEYS user:*` 没什么问题。

但生产环境默认使用 `SCAN`。

---

## 十一、常见错误

### 1. 只执行一次 SCAN 就以为遍历完成

如果返回游标不是 `0`，说明还没扫完。

### 2. 把 COUNT 当成固定返回数量

`COUNT` 是提示，不是保证。

### 3. 把 SCAN 当分页接口

`SCAN` 返回顺序不稳定，也可能重复，不适合做用户可见分页。

### 4. 扫出来就直接大批量删除

删除前要确认 key 范围、value 大小和业务影响。

---

## 十二、本节练习

请完成下面练习：

1. 写入几个 `user:profile:*` key。
2. 写入几个 `article:detail:*` key。
3. 执行 `SCAN 0`，观察返回结构。
4. 使用返回的新游标继续扫描，直到游标为 `0`。
5. 执行 `SCAN 0 MATCH user:profile:*`。
6. 执行 `SCAN 0 MATCH article:detail:* COUNT 100`。
7. 思考：为什么 `COUNT 100` 不保证返回 100 个 key？
8. 思考：为什么 `SCAN` 不适合做分页？
9. 在本地执行 `KEYS *`，感受它和 `SCAN` 的区别。
10. 思考：生产环境为什么不应该默认使用 `KEYS *`？

---

## 十三、本节小结

这一节你掌握了安全遍历命令 `SCAN`：

- `SCAN 0` 从游标 0 开始遍历。
- 返回结果包含新游标和 key 列表。
- 游标回到 `0` 表示遍历结束。
- `MATCH` 可以按模式过滤 key。
- `COUNT` 是扫描数量提示，不是精确返回数量。
- `SCAN` 更适合线上分批排查，`KEYS *` 在生产环境要慎用。

下一节我们会专门讲危险命令：`KEYS *`、`FLUSHALL`、`FLUSHDB`。这些命令不是不能知道，而是必须知道它们危险在哪里。
