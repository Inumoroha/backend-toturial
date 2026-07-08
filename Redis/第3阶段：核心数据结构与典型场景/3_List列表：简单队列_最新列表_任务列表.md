# 3. List 列表：简单队列、最新列表、任务列表

Redis List 是一个按插入顺序排列的列表结构。

它适合表达“有顺序的一组元素”，常见场景包括简单队列、最新列表、任务列表、消息缓冲等。

学完这一节后，你应该能够：

- 理解 Redis List 的左右两端操作。
- 使用 `LPUSH`、`RPUSH` 写入元素。
- 使用 `LPOP`、`RPOP` 弹出元素。
- 使用 `LRANGE` 查看列表内容。
- 使用 `LTRIM` 控制列表长度。
- 判断 List 适合和不适合的业务场景。

---

## 一、List 是什么

Redis List 可以理解为一个有序列表。

它有左端和右端：

```text
left                                      right
[ item1, item2, item3, item4, item5 ]
```

你可以从左边插入，也可以从右边插入；可以从左边弹出，也可以从右边弹出。

常见操作：

```redis
LPUSH tasks task1
RPUSH tasks task2
LPOP tasks
RPOP tasks
```

List 的元素是字符串，但可以保存 JSON 字符串。

---

## 二、常用命令总览

| 命令 | 作用 |
| --- | --- |
| `LPUSH key value [value ...]` | 从左侧插入 |
| `RPUSH key value [value ...]` | 从右侧插入 |
| `LPOP key` | 从左侧弹出 |
| `RPOP key` | 从右侧弹出 |
| `LRANGE key start stop` | 获取指定范围元素 |
| `LLEN key` | 获取列表长度 |
| `LTRIM key start stop` | 只保留指定范围元素 |
| `LINDEX key index` | 按下标读取元素 |
| `BLPOP key timeout` | 阻塞式左弹出 |
| `BRPOP key timeout` | 阻塞式右弹出 |

初学阶段重点掌握 `LPUSH`、`RPUSH`、`LPOP`、`RPOP`、`LRANGE`、`LLEN`、`LTRIM`。

---

## 三、LPUSH 和 RPUSH：写入元素

从左侧插入：

```redis
LPUSH messages msg1
LPUSH messages msg2
LPUSH messages msg3
```

查看：

```redis
LRANGE messages 0 -1
```

返回：

```text
1) "msg3"
2) "msg2"
3) "msg1"
```

因为每次都从左侧插入，新元素会排在前面。

从右侧插入：

```redis
RPUSH queue task1
RPUSH queue task2
RPUSH queue task3
```

查看：

```redis
LRANGE queue 0 -1
```

返回：

```text
1) "task1"
2) "task2"
3) "task3"
```

从右侧插入时，顺序更像普通队列的追加。

---

## 四、LPOP 和 RPOP：弹出元素

从左侧弹出：

```redis
LPOP queue
```

从右侧弹出：

```redis
RPOP queue
```

弹出意味着读取并删除。

如果列表为空，返回：

```text
(nil)
```

利用插入和弹出的方向组合，可以实现不同语义。

| 写入 | 弹出 | 效果 |
| --- | --- | --- |
| `LPUSH` | `RPOP` | 队列，先进先出 |
| `RPUSH` | `LPOP` | 队列，先进先出 |
| `LPUSH` | `LPOP` | 栈，后进先出 |
| `RPUSH` | `RPOP` | 栈，后进先出 |

实际项目里，队列更常见。

---

## 五、LRANGE：查看列表范围

`LRANGE` 用来查看指定范围的元素。

语法：

```redis
LRANGE key start stop
```

查看全部：

```redis
LRANGE queue 0 -1
```

查看前 10 个：

```redis
LRANGE queue 0 9
```

Redis List 下标从 0 开始。

`-1` 表示最后一个元素，`-2` 表示倒数第二个元素。

例如：

```redis
LRANGE queue 0 2
```

表示读取第 0 到第 2 个元素，一共最多 3 个。

注意：如果列表很长，不要随便 `LRANGE key 0 -1`，这会一次拉取全部元素。

---

## 六、LLEN：查看列表长度

```redis
LLEN queue
```

返回列表元素数量。

它常用于：

- 查看队列积压。
- 判断最新列表长度。
- 调试任务是否持续堆积。

如果任务队列长度持续增长，说明消费速度可能跟不上生产速度。

---

## 七、LTRIM：控制列表长度

`LTRIM` 用来只保留指定范围元素。

它非常适合做“最新 N 条”列表。

比如保存最新 10 条动态：

```redis
LPUSH feed:user:1001 post1
LTRIM feed:user:1001 0 9
```

每次有新动态：

```redis
LPUSH feed:user:1001 post2
LTRIM feed:user:1001 0 9
```

这样列表永远只保留 10 条。

查看：

```redis
LRANGE feed:user:1001 0 9
```

这种模式很常见：

```text
LPUSH 新内容
LTRIM 0 N-1 控制长度
```

---

## 八、场景一：简单队列

假设有任务需要异步处理。

生产者写入任务：

```redis
RPUSH task:email '{"to":"a@example.com","subject":"hello"}'
RPUSH task:email '{"to":"b@example.com","subject":"hi"}'
```

消费者取任务：

```redis
LPOP task:email
```

这形成先进先出队列：先进入的任务先被处理。

如果队列为空，`LPOP` 会立刻返回 `(nil)`。

如果希望没有任务时等待，可以使用阻塞命令：

```redis
BLPOP task:email 5
```

表示最多等待 5 秒。

---

## 九、List 做队列的局限

List 可以做简单队列，但不是完整消息队列。

局限包括：

- 消费者取出后如果处理失败，任务可能丢失。
- 不方便做消息确认 ACK。
- 不方便查看某条消息的消费状态。
- 不适合复杂重试和死信队列。
- 不适合大规模消息流治理。

如果只是本地开发、小任务、简单异步处理，List 可以。

如果业务要求可靠消息、重试、确认、消费者组，Redis Stream、RabbitMQ、Kafka 更合适。

---

## 十、场景二：最新列表

比如保存某个用户最新 10 条动态。

key：

```text
feed:user:1001
```

写入新动态：

```redis
LPUSH feed:user:1001 post:9003
LTRIM feed:user:1001 0 9
```

读取最新 10 条：

```redis
LRANGE feed:user:1001 0 9
```

这种方式适合：

- 最新文章 ID 列表。
- 最新评论 ID 列表。
- 最近访问记录。
- 最近操作日志的轻量缓存。

通常 List 里只放 ID，然后再根据 ID 批量查详情缓存或数据库。

不要把很大的完整对象都塞进最新列表。

---

## 十一、场景三：任务列表

如果你只是想维护一个待处理任务列表，可以使用 List。

写入任务 ID：

```redis
RPUSH task:resize_image image_task_1001
RPUSH task:resize_image image_task_1002
```

消费任务：

```redis
LPOP task:resize_image
```

查看积压数量：

```redis
LLEN task:resize_image
```

如果积压越来越多，说明消费者数量不足或任务处理太慢。

---

## 十二、List 不适合什么场景

List 不适合：

- 去重集合。
- 排行榜。
- 按分数排序的数据。
- 可靠消息队列。
- 按成员快速判断是否存在。
- 超大列表无限增长。

比如用户标签不适合 List，因为标签不应该重复，还需要判断是否包含某个标签。Set 更合适。

排行榜不适合 List，因为 List 没有按分数自动排序能力。ZSet 更合适。

---

## 十三、常见错误

### 1. 插入和弹出方向搞反

想做先进先出队列，却用了 `LPUSH` + `LPOP`，结果变成后进先出栈。

### 2. 最新列表不做 LTRIM

只 `LPUSH` 不 `LTRIM`，列表会无限增长。

### 3. 一次 LRANGE 全量大列表

`LRANGE key 0 -1` 对大列表可能很重。

### 4. 把 List 当可靠消息队列

List 做简单队列可以，但可靠消息要考虑 Stream 或专业 MQ。

---

## 十四、本节练习

请完成下面练习：

1. 使用 `RPUSH` 写入 3 个任务到 `task:email`。
2. 使用 `LRANGE task:email 0 -1` 查看任务顺序。
3. 使用 `LPOP` 弹出一个任务。
4. 使用 `LLEN` 查看剩余任务数量。
5. 使用 `LPUSH` 写入 12 条动态到 `feed:user:1001`。
6. 每次写入后执行 `LTRIM feed:user:1001 0 9`。
7. 使用 `LRANGE feed:user:1001 0 9` 查看最新 10 条。
8. 尝试 `LPUSH` + `LPOP`，观察它像队列还是栈。
9. 思考：List 队列如果消费者取出任务后崩溃，会发生什么？
10. 思考：为什么排行榜不适合用 List？

---

## 十五、本节小结

List 适合保存有顺序的一组元素。

你需要记住：

- `LPUSH` 从左侧插入，`RPUSH` 从右侧插入。
- `LPOP` 从左侧弹出，`RPOP` 从右侧弹出。
- 不同方向组合可以形成队列或栈。
- `LRANGE` 查看范围，`LLEN` 查看长度。
- `LTRIM` 常用于控制最新列表长度。
- List 可以做简单队列，但不等于可靠消息队列。

下一节我们学习 Set，它适合去重、抽奖、共同好友和标签集合。
