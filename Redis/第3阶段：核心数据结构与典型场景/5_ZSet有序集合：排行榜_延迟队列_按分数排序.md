# 5. ZSet 有序集合：排行榜、延迟队列、按分数排序

ZSet 是 Redis 中非常有特色的数据结构。

它和 Set 一样，成员不重复；但它比 Set 多了一个 score。Redis 会根据 score 对成员排序，因此 ZSet 非常适合排行榜、延迟队列、按分数排序的业务数据。

学完这一节后，你应该能够：

- 理解 ZSet 的 member 和 score。
- 使用 `ZADD` 添加或更新成员分数。
- 使用 `ZRANGE`、`ZREVRANGE` 获取排名列表。
- 使用 `ZSCORE`、`ZRANK`、`ZREVRANK` 查询分数和排名。
- 使用 `ZINCRBY` 增加分数。
- 理解排行榜和延迟队列的基本实现方式。
- 判断 ZSet 适合和不适合的场景。

---

## 一、ZSet 是什么

ZSet 全名是 Sorted Set，有序集合。

它的结构可以理解为：

```text
member -> score
```

例如文章热榜：

```text
article:1001 -> 99
article:1002 -> 120
article:1003 -> 75
```

Redis 会按 score 排序。

添加数据：

```redis
ZADD rank:article:daily 99 article:1001 120 article:1002 75 article:1003
```

查看分数从低到高：

```redis
ZRANGE rank:article:daily 0 -1 WITHSCORES
```

查看分数从高到低：

```redis
ZREVRANGE rank:article:daily 0 -1 WITHSCORES
```

排行榜通常使用从高到低。

---

## 二、常用命令总览

| 命令 | 作用 |
| --- | --- |
| `ZADD key score member [score member ...]` | 添加或更新成员分数 |
| `ZSCORE key member` | 查看成员分数 |
| `ZINCRBY key increment member` | 增加成员分数 |
| `ZRANGE key start stop [WITHSCORES]` | 按分数从低到高取范围 |
| `ZREVRANGE key start stop [WITHSCORES]` | 按分数从高到低取范围 |
| `ZRANK key member` | 查看从低到高排名 |
| `ZREVRANK key member` | 查看从高到低排名 |
| `ZREM key member [member ...]` | 删除成员 |
| `ZCARD key` | 成员数量 |
| `ZRANGEBYSCORE key min max` | 按分数范围查询 |
| `ZREMRANGEBYRANK key start stop` | 按排名范围删除 |
| `ZREMRANGEBYSCORE key min max` | 按分数范围删除 |

初学阶段重点掌握 `ZADD`、`ZINCRBY`、`ZREVRANGE`、`ZSCORE`、`ZREVRANK`。

---

## 三、ZADD：添加或更新分数

添加文章热度：

```redis
ZADD rank:article:daily 99 article:1001
ZADD rank:article:daily 120 article:1002
ZADD rank:article:daily 75 article:1003
```

也可以一次添加多个：

```redis
ZADD rank:article:daily 99 article:1001 120 article:1002 75 article:1003
```

如果 member 已经存在，再次 `ZADD` 会更新它的 score：

```redis
ZADD rank:article:daily 150 article:1001
```

此时 `article:1001` 的分数变成 150。

ZSet 的 member 不重复，这点和 Set 类似。

---

## 四、ZRANGE 和 ZREVRANGE：查看排名

按分数从低到高：

```redis
ZRANGE rank:article:daily 0 -1 WITHSCORES
```

按分数从高到低：

```redis
ZREVRANGE rank:article:daily 0 -1 WITHSCORES
```

取前 3 名：

```redis
ZREVRANGE rank:article:daily 0 2 WITHSCORES
```

注意下标从 0 开始，所以 `0 2` 是前三个。

如果不加 `WITHSCORES`，只返回 member。

排行榜页面通常需要 member 和 score，所以建议加上 `WITHSCORES`。

---

## 五、ZSCORE 和 ZREVRANK：查看分数和排名

查看某篇文章分数：

```redis
ZSCORE rank:article:daily article:1001
```

查看从高到低的排名：

```redis
ZREVRANK rank:article:daily article:1001
```

返回值从 0 开始。

如果返回：

```text
(integer) 0
```

说明它是第一名。

如果你要显示给用户，通常要加 1：

```text
展示排名 = Redis 返回排名 + 1
```

从低到高排名使用：

```redis
ZRANK rank:article:daily article:1001
```

排行榜一般使用 `ZREVRANK`。

---

## 六、ZINCRBY：增加分数

文章被阅读一次，热度加 1：

```redis
ZINCRBY rank:article:daily 1 article:1001
```

点赞加 5 分：

```redis
ZINCRBY rank:article:daily 5 article:1001
```

如果 member 不存在，`ZINCRBY` 会创建它，并把初始分数当作 0 后增加。

这很适合动态排行榜。

比如：

```text
阅读 +1
点赞 +5
评论 +10
收藏 +8
```

都可以转成分数增量。

---

## 七、场景一：文章热榜

key：

```text
rank:article:daily:20260627
```

用户阅读文章：

```redis
ZINCRBY rank:article:daily:20260627 1 article:1001
```

用户点赞文章：

```redis
ZINCRBY rank:article:daily:20260627 5 article:1001
```

查看前 10 名：

```redis
ZREVRANGE rank:article:daily:20260627 0 9 WITHSCORES
```

查看某篇文章排名：

```redis
ZREVRANK rank:article:daily:20260627 article:1001
```

日榜可以设置 TTL，例如保留 7 天：

```redis
EXPIRE rank:article:daily:20260627 604800
```

这样旧榜单会自动清理。

---

## 八、场景二：积分排行榜

用户积分榜：

```redis
ZADD rank:user:score 100 user:1001 180 user:1002 90 user:1003
```

增加积分：

```redis
ZINCRBY rank:user:score 20 user:1001
```

查看前 10：

```redis
ZREVRANGE rank:user:score 0 9 WITHSCORES
```

查看用户排名：

```redis
ZREVRANK rank:user:score user:1001
```

注意：如果排行榜是核心业务数据，Redis 可以承担实时排名，但分数变动记录和最终权威数据仍然建议落到数据库。

---

## 九、场景三：延迟队列基础

ZSet 可以用 score 表示任务执行时间戳。

比如任务 10 秒后执行，score 就是未来的 Unix 时间戳。

写入延迟任务：

```redis
ZADD delay:queue 1790000000 task:1001
```

消费者定期查询当前已经到期的任务：

```redis
ZRANGEBYSCORE delay:queue -inf 1790000000 LIMIT 0 10
```

取到任务后删除：

```redis
ZREM delay:queue task:1001
```

基本思路是：

```text
score <= 当前时间 的任务就是到期任务
```

但真实延迟队列还要处理并发消费者抢任务、任务执行失败重试、任务可靠性等问题。

所以 ZSet 可以实现延迟队列基础，但生产级任务队列要更谨慎。

---

## 十、按分数范围查询

`ZRANGEBYSCORE` 可以按 score 范围查询。

查询分数 80 到 120 的文章：

```redis
ZRANGEBYSCORE rank:article:daily 80 120 WITHSCORES
```

查询小于等于当前时间的延迟任务：

```redis
ZRANGEBYSCORE delay:queue -inf 1790000000
```

这个能力是 ZSet 和普通 Set 的重要区别。

Set 只能判断成员是否存在，ZSet 可以按分数排序和范围查询。

---

## 十一、控制排行榜长度

排行榜不能无限增长。

如果只保留前 1000 名，可以在更新后清理低排名数据。

按从低到高删除多余元素：

```redis
ZREMRANGEBYRANK rank:article:daily 0 -1001
```

这条命令的意思是删除排名靠后的部分，保留最高的 1000 个左右。

初学阶段不必死记这个命令，先理解：排行榜需要控制长度，否则成员会越来越多。

实际项目中也可以定时清理，而不是每次写入都清理。

---

## 十二、ZSet 不适合什么场景

ZSet 不适合：

- 只需要简单去重，没有排序需求。
- 需要保存重复成员。
- 需要先进先出队列。
- 需要复杂多字段排序。
- 成员数量无限增长且不清理。

如果只是判断用户是否点赞过，用 Set 更简单。

如果只是任务队列先进先出，用 List 或 Stream 更自然。

如果要复杂 SQL 排序、筛选、分页，关系型数据库更适合。

---

## 十三、常见错误

### 1. 排名从 0 开始

`ZREVRANK` 返回 0 表示第一名。展示给用户时通常要加 1。

### 2. 分数设计混乱

热度分数要提前设计规则，例如阅读、点赞、评论分别加多少。

### 3. 榜单不设置周期或清理

排行榜 key 如果无限增长，会越来越重。

### 4. 把 ZSet 当可靠延迟队列

ZSet 可以做延迟队列基础，但可靠性、并发抢占、失败重试要额外设计。

---

## 十四、本节练习

请完成下面练习：

1. 创建 `rank:article:daily`，加入 5 篇文章和分数。
2. 使用 `ZREVRANGE` 查看从高到低排名。
3. 使用 `ZREVRANGE key 0 2 WITHSCORES` 查看前三名。
4. 使用 `ZINCRBY` 给某篇文章增加 10 分。
5. 使用 `ZSCORE` 查看文章分数。
6. 使用 `ZREVRANK` 查看文章排名。
7. 创建 `rank:user:score` 用户积分榜。
8. 使用 `ZRANGEBYSCORE` 查询某个分数区间。
9. 用时间戳作为 score，模拟一个延迟任务。
10. 思考：点赞去重用 Set 还是 ZSet？排行榜用 Set 还是 ZSet？为什么？

---

## 十五、本节小结

ZSet 是带分数的有序集合。

你需要记住：

- ZSet 的 member 不重复，每个 member 有一个 score。
- `ZADD` 添加或更新分数。
- `ZINCRBY` 增加分数，适合动态排行榜。
- `ZREVRANGE` 常用于从高到低查看榜单。
- `ZREVRANK` 查看从高到低排名，返回值从 0 开始。
- `ZRANGEBYSCORE` 可按分数范围查询，适合延迟队列基础。
- ZSet 适合排行榜、延迟队列、按分数排序的数据。
- ZSet 不适合简单去重、先进先出队列和无限增长不清理的场景。

下一节我们会把 String、Hash、List、Set、ZSet 放在一起对比，学习如何为业务选择合适的数据结构。
