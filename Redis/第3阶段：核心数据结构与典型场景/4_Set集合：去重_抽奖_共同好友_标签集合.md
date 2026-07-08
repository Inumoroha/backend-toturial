# 4. Set 集合：去重、抽奖、共同好友、标签集合

Redis Set 是无序且不重复的集合。

它非常适合表达“某个对象拥有一组不重复的成员”，比如用户标签、点赞用户集合、抽奖候选人、共同好友、黑名单、去重记录等。

学完这一节后，你应该能够：

- 理解 Set 的无序和去重特点。
- 使用 `SADD`、`SREM`、`SMEMBERS`、`SISMEMBER` 操作集合。
- 使用 `SCARD` 统计集合元素数量。
- 使用交集、并集、差集解决共同好友等问题。
- 使用 `SRANDMEMBER`、`SPOP` 做抽样和抽奖。
- 判断 Set 适合和不适合的业务场景。

---

## 一、Set 是什么

Set 是一个无序、不重复的字符串集合。

例如用户标签：

```text
user:tags:1001 = { go, redis, mysql }
```

写入：

```redis
SADD user:tags:1001 go redis mysql
```

如果重复添加：

```redis
SADD user:tags:1001 redis
```

集合里仍然只有一个 `redis`。

这就是 Set 的核心特点：自动去重。

---

## 二、常用命令总览

| 命令 | 作用 |
| --- | --- |
| `SADD key member [member ...]` | 添加成员 |
| `SREM key member [member ...]` | 删除成员 |
| `SMEMBERS key` | 获取所有成员 |
| `SISMEMBER key member` | 判断成员是否存在 |
| `SCARD key` | 获取成员数量 |
| `SPOP key [count]` | 随机弹出成员 |
| `SRANDMEMBER key [count]` | 随机返回成员但不删除 |
| `SINTER key [key ...]` | 交集 |
| `SUNION key [key ...]` | 并集 |
| `SDIFF key [key ...]` | 差集 |

初学阶段重点掌握 `SADD`、`SREM`、`SISMEMBER`、`SCARD` 和交并差集。

---

## 三、SADD：添加成员

添加用户标签：

```redis
SADD user:tags:1001 go redis mysql
```

返回值表示新增了几个成员。

如果重复添加：

```redis
SADD user:tags:1001 redis docker
```

只有 `docker` 是新成员，所以返回 1。

查看所有成员：

```redis
SMEMBERS user:tags:1001
```

返回顺序不固定，因为 Set 是无序集合。

---

## 四、SREM：删除成员

删除一个标签：

```redis
SREM user:tags:1001 mysql
```

如果成员存在，返回 1；如果不存在，返回 0。

再次查看：

```redis
SMEMBERS user:tags:1001
```

Set 很适合做“添加、删除、判断是否存在”的集合型数据。

---

## 五、SISMEMBER：判断成员是否存在

判断用户是否有某个标签：

```redis
SISMEMBER user:tags:1001 redis
```

返回：

```text
(integer) 1
```

表示存在。

判断是否有 `python`：

```redis
SISMEMBER user:tags:1001 python
```

返回：

```text
(integer) 0
```

这个命令适合：

- 判断用户是否点赞过。
- 判断用户是否在黑名单。
- 判断某个标签是否存在。
- 判断某个任务是否已经处理过。

---

## 六、SCARD：统计集合大小

统计用户标签数量：

```redis
SCARD user:tags:1001
```

统计文章点赞人数：

```redis
SCARD article:liked_users:1001
```

如果你用 Set 保存点赞用户 ID，那么 `SCARD` 就能得到去重后的点赞人数。

---

## 七、场景一：点赞去重

需求：同一个用户对一篇文章只能点赞一次。

key：

```text
article:liked_users:1001
```

用户 2001 点赞：

```redis
SADD article:liked_users:1001 2001
```

再次点赞：

```redis
SADD article:liked_users:1001 2001
```

不会重复增加。

判断是否点赞过：

```redis
SISMEMBER article:liked_users:1001 2001
```

统计点赞人数：

```redis
SCARD article:liked_users:1001
```

取消点赞：

```redis
SREM article:liked_users:1001 2001
```

这比用 List 保存点赞用户更合适，因为 List 不会自动去重。

---

## 八、场景二：用户标签集合

用户标签天然是集合：

```redis
SADD user:tags:1001 go redis mysql
```

判断用户是否有 Redis 标签：

```redis
SISMEMBER user:tags:1001 redis
```

删除标签：

```redis
SREM user:tags:1001 mysql
```

获取标签：

```redis
SMEMBERS user:tags:1001
```

如果标签数量不大，这种方式很直接。

如果标签非常多，或者要按标签反查用户，也需要额外设计反向索引：

```text
tag:users:redis
```

里面保存拥有 `redis` 标签的用户 ID。

---

## 九、场景三：共同好友

Set 的交集非常适合做共同好友。

用户 1001 的好友：

```redis
SADD user:friends:1001 2001 2002 2003 2004
```

用户 1002 的好友：

```redis
SADD user:friends:1002 2003 2004 2005
```

求共同好友：

```redis
SINTER user:friends:1001 user:friends:1002
```

返回：

```text
1) "2003"
2) "2004"
```

Set 的交集、并集、差集是它相对 String 和 List 的重要优势。

---

## 十、交集、并集、差集

交集：两个集合都有的成员。

```redis
SINTER set1 set2
```

并集：两个集合所有成员去重后的结果。

```redis
SUNION set1 set2
```

差集：在第一个集合中，但不在后面集合中的成员。

```redis
SDIFF set1 set2
```

例子：

```redis
SADD user:follow:1001 2001 2002 2003
SADD user:follow:1002 2002 2003 2004
SINTER user:follow:1001 user:follow:1002
SUNION user:follow:1001 user:follow:1002
SDIFF user:follow:1001 user:follow:1002
```

这些命令可以做共同关注、共同好友、标签交集等。

注意：如果集合特别大，交并差集也可能比较重。生产环境要评估集合大小。

---

## 十一、场景四：抽奖

Set 可以用于抽奖候选人。

用户参与抽奖：

```redis
SADD lottery:users:20260627 1001 1002 1003 1004 1005
```

随机查看一个用户但不移除：

```redis
SRANDMEMBER lottery:users:20260627
```

随机抽出一个用户并移除：

```redis
SPOP lottery:users:20260627
```

抽 3 个并移除：

```redis
SPOP lottery:users:20260627 3
```

`SPOP` 会把成员从集合里删除，适合“不重复中奖”的抽奖。

`SRANDMEMBER` 不删除，适合随机预览或允许重复抽样的场景。

---

## 十二、Set 不适合什么场景

Set 不适合：

- 需要按分数排序的数据。
- 需要保留插入顺序的数据。
- 需要队列先进先出。
- 需要保存重复元素。
- 需要按成员附带多个字段。

比如排行榜不适合 Set，因为 Set 没有分数排序。ZSet 更合适。

最新列表不适合 Set，因为 Set 无序。List 更合适。

任务队列不适合 Set，因为 Set 不表达先进先出顺序。

---

## 十三、常见错误

### 1. 以为 SMEMBERS 有固定顺序

Set 是无序的，`SMEMBERS` 返回顺序不要依赖。

### 2. 对大集合直接 SMEMBERS

如果集合很大，一次返回所有成员会很重。可以使用 `SSCAN` 分批扫描。

### 3. 用 Set 做排行榜

Set 只能去重，不能按分数排序。排行榜应该用 ZSet。

### 4. 交集操作没有评估集合大小

大集合做交并差集可能比较耗时。

---

## 十四、本节练习

请完成下面练习：

1. 创建 `user:tags:1001`，添加 `go`、`redis`、`mysql`。
2. 重复添加 `redis`，观察返回值。
3. 使用 `SISMEMBER` 判断是否有 `redis` 标签。
4. 使用 `SREM` 删除 `mysql`。
5. 使用 `SCARD` 查看标签数量。
6. 创建两个用户的好友集合。
7. 使用 `SINTER` 找共同好友。
8. 使用 `SUNION` 找两个用户所有好友。
9. 使用 `SDIFF` 找只在第一个用户好友列表里的成员。
10. 创建抽奖集合，使用 `SRANDMEMBER` 和 `SPOP` 对比效果。

---

## 十五、本节小结

Set 是无序、不重复集合。

你需要记住：

- Set 自动去重。
- `SADD` 添加成员，`SREM` 删除成员。
- `SISMEMBER` 判断成员是否存在。
- `SCARD` 统计成员数量。
- `SINTER`、`SUNION`、`SDIFF` 可做交并差集。
- Set 适合去重、标签、点赞用户、共同好友、抽奖。
- Set 不适合排行榜、队列、最新列表和需要重复元素的场景。

下一节我们学习 ZSet，它在 Set 的基础上给每个成员增加分数，适合排行榜和按分数排序的数据。
