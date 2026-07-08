# Redis 面试题与参考答案

这份文档面向后端开发面试，尤其适合 Go 后端、Java 后端、缓存系统、分布式系统相关岗位。

答题时不要只背命令。面试官更关心：

- 你是否知道 Redis 适合解决什么问题。
- 你是否知道 Redis 的边界在哪里。
- 你是否能处理缓存一致性、热点、穿透、故障降级。
- 你是否有真实项目里的 key 设计、TTL、监控、排查经验。

---

## 一、Redis 基础认知

### 1. Redis 是什么？

Redis 是一个基于内存的高性能键值数据库，常被用于缓存、计数器、排行榜、分布式锁、限流、消息队列、会话存储等场景。

它支持多种数据结构，包括 String、Hash、List、Set、Sorted Set、Bitmap、HyperLogLog、Geo、Stream 等。

面试回答可以强调：

- Redis 主要数据在内存中，所以读写很快。
- Redis 支持丰富的数据结构，不只是简单 key-value。
- Redis 通常不作为核心关系数据的唯一数据源，而是作为缓存、加速层或辅助状态系统。
- Redis 支持持久化、高可用、主从复制、哨兵和集群。

---

### 2. Redis 为什么快？

Redis 快主要因为：

1. 数据主要存储在内存中。
2. 单线程执行命令，避免了多线程锁竞争和上下文切换。
3. 使用 I/O 多路复用处理大量连接。
4. 数据结构实现高效。
5. 命令大多是简单内存操作。

补充说明：

Redis 的“单线程”主要指命令执行线程。Redis 后来的版本也引入了后台线程处理部分 I/O、关闭文件、异步释放等任务，但核心命令执行仍然强调单线程模型。

面试官追问时可以说：

单线程并不代表不能高并发。Redis 使用事件循环和 I/O 多路复用在一个线程中处理大量客户端请求，避免了复杂锁竞争。真正会拖慢 Redis 的通常是慢命令、大 key、网络延迟、持久化 fork、内存压力、Lua 脚本过慢等。

---

### 3. Redis 是单线程还是多线程？

经典回答：

Redis 的核心命令执行是单线程的，但 Redis 整体不是只有一个线程。

具体来说：

- 命令解析、执行、返回结果的核心逻辑通常在主线程。
- 后台会有线程处理 AOF 刷盘、异步删除、关闭文件等任务。
- Redis 6 以后支持 I/O 多线程，用于网络读写优化，但命令执行仍然主要由主线程完成。

为什么这样设计？

- 单线程模型简化数据结构访问，不需要大量锁。
- Redis 的瓶颈很多时候不是 CPU，而是内存和网络。
- 单线程命令执行让每条命令具有天然原子性。

---

### 4. Redis 和 Memcached 有什么区别？

可以从几个角度回答：

| 对比项 | Redis | Memcached |
| --- | --- | --- |
| 数据结构 | 丰富，支持 String、Hash、List、Set、ZSet 等 | 主要是简单 key-value |
| 持久化 | 支持 RDB、AOF | 通常不支持持久化 |
| 高可用 | 支持复制、哨兵、集群 | 能力较弱 |
| 原子操作 | 支持丰富原子命令和 Lua | 相对简单 |
| 使用场景 | 缓存、计数器、排行榜、锁、限流、消息等 | 主要缓存 |

总结：

如果只是简单缓存，两者都可以；如果需要复杂数据结构、持久化、排行榜、限流、分布式锁等能力，Redis 更常用。

---

### 5. Redis 适合放什么数据？

适合放：

- 高频读取、允许短时间不一致的数据。
- 可重建的数据。
- 计数器、排行榜、限流状态。
- 会话、验证码、临时 token。
- 热点对象详情。
- 分布式协调中的临时状态。

不适合放：

- 强一致核心交易数据。
- 不能丢失且没有落库的数据。
- 超大 value。
- 无 TTL、无限增长的数据。
- 复杂关系查询数据。

一句话：

Redis 适合做加速层和辅助状态系统，不适合作为所有核心业务数据的唯一事实来源。

---

## 二、数据结构与使用场景

### 6. Redis 有哪些常用数据结构？

常用数据结构：

- String：字符串、整数、二进制数据。
- Hash：对象字段。
- List：列表、队列。
- Set：去重集合。
- Sorted Set：排行榜、排序集合。
- Bitmap：位图统计。
- HyperLogLog：基数估算。
- Geo：地理位置。
- Stream：消息流。

答题时最好结合场景，不要只背名字。

---

### 7. String 可以用来做什么？

String 是最基础的数据类型，可以存字符串、数字、JSON、二进制内容。

常见场景：

- 缓存对象 JSON。
- 计数器。
- 分布式锁。
- 验证码。
- token。
- 限流计数。

常用命令：

```redis
SET key value EX 60
GET key
INCR key
DECR key
SETNX key value
MGET key1 key2
```

注意：

String value 不应该太大。过大的 value 会造成网络传输、内存分配、阻塞删除等问题。

---

### 8. Hash 适合什么场景？

Hash 适合存储对象的多个字段。

例如用户信息：

```redis
HSET user:1001 name "Tom" age 20 city "Shanghai"
HGET user:1001 name
HMGET user:1001 name age
HGETALL user:1001
```

适合场景：

- 用户资料。
- 商品简要信息。
- 配置项。
- 对象字段局部更新。

优点：

- 可以单独读写某个字段。
- 避免整个 JSON 反序列化和重写。

缺点：

- 对象结构复杂时不如 JSON 直观。
- field 数量过多也会形成大 key。
- 不适合复杂嵌套对象。

---

### 9. List 适合什么场景？

List 是有序列表，可以从两端插入和弹出。

常见场景：

- 简单队列。
- 最新消息列表。
- 时间线。
- 任务列表。

常用命令：

```redis
LPUSH queue task1
RPOP queue
BRPOP queue 5
LRANGE list 0 9
```

注意：

List 可以做简单队列，但如果需要消费组、确认、重试、积压管理，Redis Stream 或专业消息队列更合适。

---

### 10. Set 适合什么场景？

Set 是无序去重集合。

常见场景：

- 用户标签。
- 点赞用户集合。
- 黑名单。
- 好友关系。
- 去重统计。

常用命令：

```redis
SADD article:1:likes user1
SISMEMBER article:1:likes user1
SCARD article:1:likes
SINTER set1 set2
SUNION set1 set2
SDIFF set1 set2
```

适合需要去重和集合运算的场景。

注意：

如果集合非常大，要避免一次性 `SMEMBERS` 拉取全部元素，可以使用 `SSCAN`。

---

### 11. ZSet 适合什么场景？

ZSet 是有序集合，每个 member 有一个 score。

常见场景：

- 排行榜。
- 热门文章。
- 延迟队列。
- 按时间排序的动态列表。
- 滑动窗口限流。

常用命令：

```redis
ZADD rank 100 user1
ZINCRBY rank 10 user1
ZREVRANGE rank 0 9 WITHSCORES
ZRANK rank user1
ZREM rank user1
```

回答时可以举例：

文章热榜可以用 `ZINCRBY article:hot 1 article_id` 增加热度，用 `ZREVRANGE article:hot 0 9 WITHSCORES` 查询 Top 10。

---

### 12. Bitmap 适合什么场景？

Bitmap 本质上是对 String 的 bit 位操作。

适合：

- 用户签到。
- 是否访问过。
- 活跃用户标记。
- 布尔状态统计。

示例：

```redis
SETBIT sign:2026:07 1001 1
GETBIT sign:2026:07 1001
BITCOUNT sign:2026:07
```

优点：

- 空间非常省。
- 适合大量布尔状态。

缺点：

- 适合 ID 稠密或可控的场景。
- 如果 offset 极大且稀疏，空间可能被拉大。

---

### 13. HyperLogLog 适合什么场景？

HyperLogLog 用于基数估算，比如 UV 统计。

示例：

```redis
PFADD page:uv user1 user2 user3
PFCOUNT page:uv
PFMERGE total:uv page1:uv page2:uv
```

特点：

- 占用内存很小。
- 结果是近似值，不是精确值。
- 适合大规模 UV 统计。

不适合：

- 需要精确统计的业务。
- 需要知道具体有哪些用户的场景。

---

### 14. Stream 适合什么场景？

Stream 是 Redis 的消息流结构。

适合：

- 轻量消息队列。
- 访问日志。
- 异步事件。
- 消费组场景。

常用命令：

```redis
XADD visit_stream * code aB9xK2 ip 127.0.0.1
XREAD COUNT 10 STREAMS visit_stream 0
XGROUP CREATE visit_stream group1 0 MKSTREAM
XREADGROUP GROUP group1 consumer1 COUNT 10 STREAMS visit_stream >
XACK visit_stream group1 message_id
```

注意：

Stream 要考虑：

- 消息确认。
- Pending List。
- 重试。
- 死信。
- 裁剪。
- 消费延迟监控。

如果业务是重型消息系统，Kafka、RabbitMQ、RocketMQ 可能更合适。

---

## 三、Key 设计与 TTL

### 15. Redis key 如何设计？

好的 key 设计应该：

- 有业务前缀。
- 层级清晰。
- 可读性好。
- 长度适中。
- 避免包含巨大字符串。
- 方便定位和排查。

示例：

```text
user:profile:{user_id}
article:detail:{article_id}
shortlink:detail:{code}
shortlink:visit_count:{code}
shortlink:create_limit:{user_id}
```

不推荐：

```text
data
cache
user
very:very:very:long:url:https://...
```

补充：

Redis Cluster 中如果需要多个 key 落到同一槽位，可以使用 Hash Tag：

```text
order:{1001}:info
order:{1001}:items
```

---

### 16. TTL 应该如何设计？

TTL 要根据数据类型和业务要求设计。

常见策略：

| 数据 | TTL 建议 |
| --- | --- |
| 短期验证码 | 1 到 5 分钟 |
| 登录 token | 根据登录有效期 |
| 对象详情缓存 | 5 分钟到数小时 |
| 空值缓存 | 30 秒到 2 分钟 |
| 限流 key | 一个限流窗口 |
| 排行榜日榜 | 保留数天到数月 |

注意：

- 不要所有 key 都永不过期。
- 不要大量 key 同一时刻过期。
- 热点缓存可以加随机抖动。
- 空值缓存 TTL 要短。

---

### 17. 为什么要给 TTL 加随机抖动？

如果大量 key 在同一时间过期，会导致大量请求同时回源数据库，形成缓存雪崩。

解决方式之一是 TTL 加随机抖动。

示例：

```text
基础 TTL：30 分钟
随机抖动：0 到 5 分钟
最终 TTL：30 到 35 分钟
```

这样 key 不会集中在同一秒失效。

---

### 18. 什么是大 key？有什么危害？

大 key 指 value 很大，或者集合元素很多的 key。

例如：

- 一个 String value 几 MB。
- 一个 Hash 有几十万个 field。
- 一个 List 有百万个元素。
- 一个 Set / ZSet 里元素很多。

危害：

- 网络传输慢。
- 序列化和反序列化慢。
- 删除可能阻塞 Redis。
- 迁移和持久化成本高。
- 集群节点数据倾斜。

治理方式：

- 拆分 key。
- 限制 value 大小。
- 使用 `SCAN` 系列命令分批处理。
- 删除大 key 使用 `UNLINK`。
- 监控 big key。

---

### 19. 什么是热 key？有什么危害？

热 key 是访问频率特别高的 key。

危害：

- 单个 Redis 节点压力过大。
- 网络出口被打满。
- Redis CPU 升高。
- 在 Cluster 中造成单槽位热点。

治理方式：

- 本地缓存。
- 多副本缓存。
- 热 key 拆分。
- 请求合并。
- SingleFlight 防止重复回源。
- 预热缓存。
- 限流保护。

---

## 四、缓存设计

### 20. 什么是 Cache-Aside Pattern？

Cache-Aside 是最常见的缓存模式。

读流程：

```text
先读缓存
缓存命中 -> 返回
缓存未命中 -> 查数据库
数据库存在 -> 写缓存 -> 返回
数据库不存在 -> 写空值缓存或返回空
```

写流程：

```text
先更新数据库
再删除缓存
```

优点：

- 简单。
- 易理解。
- 适合大多数读多写少业务。

缺点：

- 可能出现短暂不一致。
- 需要处理缓存击穿、穿透、雪崩。

---

### 21. 为什么更新数据时通常是删除缓存，而不是更新缓存？

推荐做法：

```text
更新数据库 -> 删除缓存
```

原因：

- 缓存可能只是数据库数据的一个派生视图。
- 更新缓存容易遗漏字段或逻辑。
- 多处写入时容易出现并发覆盖。
- 删除缓存更简单，下一次读取会重新从数据库加载。

注意：

删除缓存失败仍然会导致旧缓存短时间存在，所以要配合 TTL、重试、日志、消息队列补偿等机制。

---

### 22. 先更新数据库再删除缓存，还是先删除缓存再更新数据库？

通常推荐：

```text
先更新数据库，再删除缓存
```

如果先删缓存再更新数据库，可能发生：

```text
1. 请求 A 删除缓存
2. 请求 B 读缓存未命中
3. 请求 B 查到数据库旧值
4. 请求 B 把旧值写回缓存
5. 请求 A 更新数据库为新值
```

最终缓存里可能是旧值。

先更新数据库再删除缓存也不是绝对完美，但风险更小，更常作为默认方案。

---

### 23. 删除缓存失败怎么办？

可以采用多种补偿手段：

1. 缓存设置合理 TTL。
2. 删除失败记录日志和告警。
3. 短时间重试删除。
4. 通过消息队列异步重试。
5. 监听 binlog / CDC 删除缓存。
6. 延迟双删。

面试可以这样答：

项目里一般不会只依赖一次删除。对于一致性要求普通的缓存，用 TTL 兜底即可；对于一致性要求较高的缓存，会增加删除失败重试、消息队列补偿，甚至通过 binlog 订阅做缓存失效。

---

### 24. 什么是缓存穿透？如何解决？

缓存穿透是指请求查询一个缓存和数据库都不存在的数据，导致每次请求都打到数据库。

典型场景：

- 恶意请求不存在的 ID。
- 随机短码攻击。
- 查询已经删除的数据。

解决方案：

1. 参数校验。
2. 空值缓存。
3. 布隆过滤器。
4. 接口限流。
5. 黑名单策略。

空值缓存示例：

```text
user:not_found:1001 -> 1，TTL 60 秒
```

注意：

空值缓存 TTL 要短，防止数据新建后短时间仍然返回不存在。

---

### 25. 什么是缓存击穿？如何解决？

缓存击穿是指某个热点 key 过期瞬间，大量请求同时打到数据库。

解决方案：

1. 热点 key 不过期或逻辑过期。
2. 互斥锁，只允许一个请求回源。
3. SingleFlight 合并请求。
4. 缓存预热。
5. TTL 随机抖动。

示例回答：

热点商品详情过期时，可以让第一个请求拿锁回源数据库并重建缓存，其他请求等待或返回旧值，避免数据库被打爆。

---

### 26. 什么是缓存雪崩？如何解决？

缓存雪崩是指大量缓存同时失效，导致请求集中打到数据库。

解决方案：

1. TTL 加随机抖动。
2. 分批预热。
3. 多级缓存。
4. 限流降级。
5. 熔断保护。
6. Redis 高可用。

区别：

- 穿透：查不存在的数据。
- 击穿：热点 key 失效。
- 雪崩：大量 key 同时失效或 Redis 整体不可用。

---

### 27. 什么是缓存预热？

缓存预热是指系统启动或活动开始前，提前把热点数据加载到 Redis。

适合：

- 秒杀商品。
- 首页配置。
- 热门文章。
- 活动页数据。

好处：

- 避免冷启动时大量请求回源数据库。
- 提高首批请求性能。

注意：

预热要控制速度，避免预热任务本身打爆数据库。

---

### 28. 什么是逻辑过期？

逻辑过期是指 Redis key 本身不设置物理过期，value 中保存一个过期时间字段。

读取时：

```text
如果未逻辑过期 -> 直接返回
如果已逻辑过期 -> 先返回旧值，再异步刷新缓存
```

优点：

- 避免热点 key 物理过期导致击穿。
- 用户请求不会阻塞等待回源。

缺点：

- 可能短时间返回旧数据。
- 实现复杂一些。

适合：

- 热点商品。
- 首页配置。
- 对一致性要求不高但可用性要求高的数据。

---

## 五、持久化与内存管理

### 29. Redis 持久化方式有哪些？

主要有：

- RDB：生成某一时刻的数据快照。
- AOF：追加记录写命令日志。
- 混合持久化：AOF 文件中包含 RDB 前缀加增量 AOF。

RDB 适合备份和快速恢复。

AOF 数据丢失更少，但文件可能更大，恢复相对慢。

生产中常结合使用。

---

### 30. RDB 的优缺点是什么？

优点：

- 文件紧凑。
- 适合备份。
- 恢复速度较快。
- 对主线程影响相对小，因为通过 fork 子进程生成快照。

缺点：

- 两次快照之间的数据可能丢失。
- fork 可能带来内存和延迟抖动。
- 数据量大时生成快照成本高。

---

### 31. AOF 的优缺点是什么？

优点：

- 数据安全性更高。
- 可以配置每秒 fsync。
- 文件是写命令日志，可读性较好。

缺点：

- 文件可能较大。
- 恢复需要重放命令，可能慢。
- AOF 重写会带来额外开销。

常见配置：

```text
appendfsync always    最安全，性能最低
appendfsync everysec  常用，最多丢约 1 秒数据
appendfsync no        依赖操作系统刷盘
```

---

### 32. 什么是 AOF 重写？

AOF 重写是把冗余写命令压缩成更小的等价数据集。

例如原来有：

```redis
SET count 1
INCR count
INCR count
INCR count
```

重写后可能变成：

```redis
SET count 4
```

目的：

- 减小 AOF 文件。
- 加快恢复速度。

注意：

AOF 重写会 fork 子进程，数据量大时可能带来内存和延迟抖动。

---

### 33. Redis 的过期删除策略是什么？

Redis 主要结合两种方式：

1. 惰性删除：访问 key 时发现过期，再删除。
2. 定期删除：周期性抽样检查过期 key 并删除。

为什么不每个 key 到期立即删除？

因为为每个 key 维护定时器成本太高。

注意：

过期 key 不一定在过期瞬间立刻消失，但对客户端访问来说会被视为不存在。

---

### 34. Redis 内存淘汰策略有哪些？

常见策略：

- `noeviction`：不淘汰，写入报错。
- `allkeys-lru`：所有 key 中淘汰最近最少使用。
- `volatile-lru`：有 TTL 的 key 中淘汰最近最少使用。
- `allkeys-lfu`：所有 key 中淘汰最不常使用。
- `volatile-lfu`：有 TTL 的 key 中淘汰最不常使用。
- `allkeys-random`：所有 key 中随机淘汰。
- `volatile-random`：有 TTL 的 key 中随机淘汰。
- `volatile-ttl`：有 TTL 的 key 中优先淘汰快过期的。

选择建议：

- 纯缓存场景常用 `allkeys-lru` 或 `allkeys-lfu`。
- 如果不希望 Redis 自动淘汰重要 key，可以使用 `noeviction`，但要处理写入失败。

---

### 35. Redis 内存满了会怎样？

取决于 `maxmemory-policy`。

如果是 `noeviction`：

- 写命令会失败。
- 读命令通常仍可执行。

如果是淘汰策略：

- Redis 会根据策略淘汰部分 key。

排查方式：

- 查看 `INFO memory`。
- 查看 `used_memory`。
- 查看 `evicted_keys`。
- 分析 big key。
- 检查 TTL 设计。
- 检查是否有无限增长集合。

---

## 六、分布式锁

### 36. 如何用 Redis 实现分布式锁？

基础加锁：

```redis
SET lock:order:1001 request_id NX EX 10
```

含义：

- `NX`：key 不存在时才设置。
- `EX 10`：设置 10 秒过期。
- `request_id`：锁持有者标识。

释放锁要用 Lua 判断 value 后删除：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

不能直接 `DEL lock`，否则可能误删别人的锁。

---

### 37. 分布式锁为什么要设置过期时间？

如果不设置过期时间，持有锁的进程宕机后，锁会永久存在，其他请求永远拿不到锁。

过期时间是防死锁兜底。

但过期时间也带来问题：

如果业务执行时间超过锁过期时间，锁可能被别人拿走，导致并发执行。

解决方式：

- 合理设置过期时间。
- 锁续期。
- 看门狗机制。
- 保证业务幂等。
- 减小锁粒度。

---

### 38. Redis 分布式锁有哪些风险？

风险：

- 锁过期但业务没执行完。
- 释放锁时误删别人锁。
- Redis 主从切换导致锁丢失。
- 客户端长时间 STW 或网络暂停。
- 锁粒度过大导致吞吐下降。

防护：

- value 使用唯一 request id。
- Lua 解锁。
- 设置合理过期时间。
- 必要时续期。
- 业务幂等兜底。
- 对强一致场景谨慎使用 Redis 锁。

---

### 39. 什么是 Redlock？

Redlock 是 Redis 作者提出的一种基于多个 Redis 独立实例的分布式锁算法。

大致思路：

- 客户端尝试在多个 Redis 节点加锁。
- 超过半数节点加锁成功，并且总耗时小于锁有效期，则认为加锁成功。

争议：

- 在网络分区、时钟漂移、进程暂停等复杂情况下，安全性存在争议。
- 对强一致要求很高的场景，可能需要 ZooKeeper、etcd、数据库事务等更强协调机制。

面试回答：

普通业务防重复执行可以使用 Redis 锁；涉及强一致、资金、库存核心扣减时，需要结合数据库约束、事务、幂等，不能只依赖 Redis 锁。

---

## 七、限流与计数器

### 40. 如何用 Redis 实现固定窗口限流？

例如用户每分钟最多请求 100 次。

key：

```text
limit:user:{user_id}:{minute}
```

命令：

```redis
INCR limit:user:1001:202607051200
EXPIRE limit:user:1001:202607051200 60
```

或者用一个 key：

```redis
INCR limit:user:1001
EXPIRE limit:user:1001 60
```

超过阈值就拒绝。

缺点：

- 窗口边界可能突刺。

---

### 41. 如何用 Redis 实现滑动窗口限流？

可以用 ZSet。

思路：

```text
1. 删除窗口外的请求记录
2. 统计窗口内请求数量
3. 如果数量超过限制，拒绝
4. 否则写入当前请求时间
5. 设置 key 过期
```

命令思路：

```redis
ZREMRANGEBYSCORE limit:user:1001 0 now-window
ZCARD limit:user:1001
ZADD limit:user:1001 now request_id
EXPIRE limit:user:1001 60
```

为了保证原子性，通常用 Lua 包起来。

---

### 42. 计数器如何实现？

Redis String 可以直接使用 `INCR`。

示例：

```redis
INCR article:read_count:1001
INCRBY article:read_count:1001 10
GET article:read_count:1001
```

优点：

- 原子。
- 性能高。
- 实现简单。

注意：

- 如果计数重要，需要定期同步到数据库。
- Redis 故障时要考虑是否允许丢计数。
- 热点计数器可能形成热 key。

---

### 43. 如何处理热点计数器？

例如热门文章阅读数极高，一个 key 被频繁 `INCR`。

处理方式：

- 分片计数：`article:read_count:{id}:{shard}`。
- 本地聚合后批量写 Redis。
- 异步日志流聚合。
- 限制统计精度，允许延迟。
- 使用队列削峰。

查询时汇总多个 shard。

---

## 八、Lua 与原子性

### 44. Redis 命令是原子的吗？

单条 Redis 命令是原子执行的。

例如：

```redis
INCR count
```

不会被其他命令打断。

但多条命令组合不一定原子。

例如：

```redis
INCR key
EXPIRE key 60
```

中间如果客户端宕机，可能导致 key 没有过期时间。

这种场景可以使用 Lua 保证多步操作原子执行。

---

### 45. Lua 脚本有什么作用？

Lua 可以把多个 Redis 命令打包成一个原子操作。

适合：

- 分布式锁释放。
- 限流。
- 库存扣减。
- 条件更新。
- 多命令组合。

注意：

- Lua 执行期间会阻塞 Redis 主线程。
- 脚本不能太复杂。
- 不要在 Lua 里做长循环或大 key 操作。
- 要控制执行时间。

---

### 46. Lua 脚本为什么会阻塞 Redis？

Redis 执行命令主要是单线程的。

Lua 脚本作为一个整体执行，执行期间其他命令要等待。

如果 Lua 脚本很慢，会导致 Redis 整体响应变慢。

所以 Lua 应该只用于短小的原子逻辑，不适合写复杂业务。

---

## 九、事务、Pipeline 与发布订阅

### 47. Redis 事务是什么？

Redis 事务使用：

```redis
MULTI
SET a 1
INCR b
EXEC
```

特点：

- 命令会排队。
- `EXEC` 时按顺序执行。
- 不支持传统关系数据库那种完整回滚。

如果命令执行时出错，其他命令可能仍然执行。

Redis 事务更像命令批量顺序执行机制，而不是强事务系统。

---

### 48. Redis Pipeline 是什么？

Pipeline 是客户端把多条命令一次性发送给 Redis，减少网络往返。

示例场景：

- 批量读取多个 key。
- 批量写入缓存。
- 统计接口同时读取详情和计数。

注意：

- Pipeline 不是事务。
- 不保证原子性。
- 只是减少 RTT，提高吞吐。

---

### 49. Pipeline 和 MGET 有什么区别？

`MGET` 是 Redis 的一个命令，一次读取多个 String key。

Pipeline 是客户端层面的批量发送，可以包含不同命令。

例如 Pipeline 可以同时执行：

```redis
GET key1
HGET key2 field
ZREVRANGE rank 0 9
```

如果只是批量读取 String，`MGET` 更简单。

如果命令类型不同，Pipeline 更通用。

---

### 50. Redis Pub/Sub 有什么问题？

Pub/Sub 是发布订阅机制。

特点：

- 实时推送。
- 消息不会持久化。
- 消费者断开期间消息会丢失。
- 没有确认和重试机制。

适合：

- 简单通知。
- 缓存失效广播。
- 实时性强但允许丢的消息。

不适合：

- 可靠任务队列。
- 订单事件。
- 必须消费成功的业务。

可靠消息更适合 Stream 或专业消息队列。

---

## 十、高可用、复制与集群

### 51. Redis 主从复制的作用是什么？

作用：

- 数据冗余。
- 读写分离。
- 故障恢复基础。
- 支持哨兵故障转移。

主节点负责写，从节点复制数据。

注意：

- Redis 主从复制是异步的。
- 主节点写入成功不代表从节点一定已经同步。
- 主从切换可能丢少量数据。

---

### 52. Redis 主从复制大致流程是什么？

大致流程：

1. 从节点连接主节点。
2. 发送同步请求。
3. 主节点生成 RDB 快照。
4. 主节点把快照发送给从节点。
5. 从节点加载快照。
6. 主节点继续把增量写命令发送给从节点。

Redis 支持部分重同步，避免每次断线都全量同步。

---

### 53. 什么是哨兵 Sentinel？

Sentinel 用于 Redis 主从架构的高可用。

主要功能：

- 监控主从节点。
- 判断主节点是否下线。
- 自动选举新的主节点。
- 通知客户端新的主节点地址。

注意：

Sentinel 解决的是主从高可用，不解决数据分片扩容。

---

### 54. 什么是 Redis Cluster？

Redis Cluster 是 Redis 的分布式集群方案。

特点：

- 数据按槽位分片。
- 总共有 16384 个 hash slot。
- 每个 key 根据 hash 结果落到某个槽。
- 支持多个主节点分摊数据和流量。
- 每个主节点可以有从节点。

适合：

- 数据量较大。
- 单机内存不够。
- QPS 需要多节点分摊。

---

### 55. Redis Cluster 为什么有 16384 个槽？

Redis Cluster 使用 hash slot 管理数据分片。

key 会被映射到 0 到 16383 的槽位。

槽位再分配给不同主节点。

好处：

- 数据迁移以槽为单位。
- 扩容缩容时移动槽位即可。
- 客户端可以缓存槽位到节点的映射。

---

### 56. Redis Cluster 下多 key 操作有什么限制？

多 key 操作要求这些 key 在同一个槽位。

例如：

```redis
MGET key1 key2
```

如果 `key1` 和 `key2` 不在同一槽位，Cluster 会报错。

解决方式：

使用 Hash Tag：

```text
user:{1001}:profile
user:{1001}:settings
```

大括号内的内容相同，就会落到同一槽位。

注意：

不要滥用 Hash Tag，否则会造成槽位热点。

---

### 57. Redis Cluster 的 MOVED 和 ASK 是什么？

`MOVED`：

- 表示 key 所在槽位已经归属另一个节点。
- 客户端应该更新槽位映射。

`ASK`：

- 表示槽位正在迁移中。
- 客户端临时向目标节点请求。

面试中可以说：

成熟 Redis Cluster 客户端通常会自动处理 MOVED 和 ASK。

---

## 十一、性能排查

### 58. Redis 变慢了怎么排查？

可以按层次排查：

1. 看应用日志：哪些接口慢，哪些命令慢。
2. 看 Redis `INFO`：CPU、内存、连接数、命中率、淘汰 key。
3. 看 `SLOWLOG GET`：是否有慢命令。
4. 看 big key 和 hot key。
5. 看网络延迟。
6. 看连接池是否耗尽。
7. 看持久化是否 fork 或 AOF rewrite。
8. 看是否有主从同步、集群迁移。
9. 看最近是否上线新功能或流量突增。

答题重点：

不要一看到 Redis 超时就认定 Redis 本身有问题，也可能是客户端连接池、网络、业务访问模式或数据库回源导致。

---

### 59. 常用 Redis 排查命令有哪些？

常用命令：

```redis
INFO
SLOWLOG GET
CLIENT LIST
MEMORY STATS
MEMORY USAGE key
TTL key
SCAN
LATENCY DOCTOR
CONFIG GET maxmemory
```

谨慎使用：

```redis
KEYS *
MONITOR
HGETALL big_hash
SMEMBERS big_set
LRANGE big_list 0 -1
```

这些命令在生产可能造成阻塞或巨大输出。

---

### 60. SLOWLOG 能查到什么？

`SLOWLOG` 记录执行时间超过阈值的慢命令。

常用：

```redis
SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET
```

能帮助发现：

- 大 key 操作。
- 慢 Lua。
- 大范围查询。
- 不合理命令。

注意：

SLOWLOG 记录的是命令执行时间，不包括网络传输和客户端排队时间。

如果应用看到 Redis 很慢，但 SLOWLOG 没有慢命令，可能是网络、连接池、客户端排队、Redis CPU 饱和等原因。

---

### 61. Redis 命中率下降怎么排查？

排查方向：

- 是否新版本修改了 key 命名。
- 是否大量 key 过期。
- 是否内存不足导致淘汰。
- 是否缓存写入失败。
- 是否 TTL 太短。
- 是否请求访问大量冷数据。
- 是否 Redis 重启导致缓存清空。
- 是否缓存预热没有执行。

关注指标：

```text
keyspace_hits
keyspace_misses
evicted_keys
expired_keys
used_memory
数据库 QPS
```

命中率下降的最大风险是数据库回源压力暴增。

---

### 62. 如何发现 big key？

方式：

- `redis-cli --bigkeys`
- `MEMORY USAGE key`
- `SCAN` 分批扫描后抽样检查。
- 监控系统统计。
- RDB 离线分析。

注意：

不要在生产使用 `KEYS *` 全量扫描。

对于集合类型，不要直接 `HGETALL`、`SMEMBERS`、`LRANGE 0 -1`。

---

### 63. 如何安全删除大 key？

不推荐直接：

```redis
DEL big_key
```

因为可能阻塞 Redis。

推荐：

```redis
UNLINK big_key
```

`UNLINK` 会异步释放内存。

对于超大集合，也可以分批删除元素：

- Hash 用 `HSCAN` + `HDEL`。
- Set 用 `SSCAN` + `SREM`。
- ZSet 用 `ZSCAN` + `ZREM`。
- List 可以分段处理或重建 key。

---

### 64. Redis 连接数过高怎么排查？

排查：

- `CLIENT LIST` 查看客户端来源。
- 应用是否每个请求创建 Redis client。
- 连接池配置是否合理。
- 是否有连接泄漏。
- 是否有短连接风暴。
- 是否有大量空闲连接。

Go 项目中要注意：

- `redis.Client` 应该复用。
- 不要每个请求 `redis.NewClient`。
- 设置合理 `PoolSize`、超时和最大空闲连接。

---

## 十二、安全、监控与生产治理

### 65. Redis 生产环境有哪些安全建议？

建议：

- 不暴露公网。
- 使用内网访问。
- 开启认证。
- 使用 ACL。
- 限制危险命令。
- 配置 `bind` 和 `protected-mode`。
- 必要时启用 TLS。
- 密码不要写入代码仓库。
- 备份和配置纳入管理。
- 开启监控和告警。

危险命令包括：

```redis
FLUSHALL
FLUSHDB
CONFIG
SHUTDOWN
DEBUG
MODULE
KEYS
MONITOR
```

---

### 66. Redis ACL 有什么作用？

ACL 用来控制：

- 用户是否启用。
- 密码。
- 可访问 key 前缀。
- 可执行命令。

示例：

```redis
ACL SETUSER shortlink_app on >app-password ~shortlink:* +@read +@write -flushall -flushdb -config
```

好处：

- 不同应用不同账号。
- 限制 key 范围。
- 禁止危险命令。
- 降低凭证泄露影响面。

---

### 67. Redis 需要监控哪些指标？

常见指标：

- QPS。
- P95/P99 延迟。
- 命中率。
- used_memory。
- maxmemory。
- connected_clients。
- blocked_clients。
- rejected_connections。
- expired_keys。
- evicted_keys。
- keyspace_hits / misses。
- slowlog。
- replication lag。
- AOF / RDB 状态。
- CPU 使用率。

告警重点：

- 内存接近上限。
- 淘汰 key 持续增长。
- 命中率骤降。
- 慢命令增加。
- 连接数异常增长。
- 主从延迟过高。
- 持久化失败。

---

### 68. Redis 故障时业务如何降级？

不同能力降级方式不同。

示例：

| Redis 能力 | 故障时策略 |
| --- | --- |
| 详情缓存 | 回源数据库 |
| 访问计数 | 暂时丢弃或本地聚合 |
| 排行榜 | 返回旧榜或空榜 |
| 限流 | 写接口保守拒绝，读接口可放行 |
| 分布式锁 | 拒绝关键操作或走数据库幂等 |
| 会话 | 可能要求重新登录 |
| 验证码 | 暂停发送或本地保护 |

核心原则：

缓存失败不一定等于业务失败。

但限流、锁、验证码这类保护性能力失败时，要更保守。

---

## 十三、Go 接入 Redis

### 69. Go 项目如何接入 Redis？

常用客户端：

```go
github.com/redis/go-redis/v9
```

初始化示例：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "127.0.0.1:6379",
    Username:     "app",
    Password:     "password",
    DB:           0,
    PoolSize:     20,
    MinIdleConns: 5,
    DialTimeout:  2 * time.Second,
    ReadTimeout:  500 * time.Millisecond,
    WriteTimeout: 500 * time.Millisecond,
})
```

注意：

- Client 要复用。
- 每次操作带 context。
- 设置超时。
- 区分 `redis.Nil`。
- 不要把密码写死。

---

### 70. Go 中如何处理 redis.Nil？

`redis.Nil` 表示 key 不存在，不是系统错误。

示例：

```go
val, err := rdb.Get(ctx, key).Result()
if errors.Is(err, redis.Nil) {
    return "", nil
}
if err != nil {
    return "", err
}
return val, nil
```

如果把 `redis.Nil` 当 500，会导致缓存未命中时接口错误。

---

### 71. Go Redis Client 为什么不能每个请求创建？

因为 `redis.Client` 内部管理连接池。

每个请求创建 client 会导致：

- 连接数暴涨。
- 频繁创建 TCP 连接。
- 性能下降。
- Redis 连接被打满。

正确做法：

- 应用启动时创建一个 client。
- 全局复用。
- 关闭应用时统一 Close。

---

### 72. Go 中 Redis 操作如何设置超时？

使用 context：

```go
ctx, cancel := context.WithTimeout(r.Context(), 300*time.Millisecond)
defer cancel()

val, err := rdb.Get(ctx, key).Result()
```

同时配置客户端读写超时：

```go
ReadTimeout:  500 * time.Millisecond
WriteTimeout: 500 * time.Millisecond
```

不要让 Redis 调用无限等待。

---

## 十四、项目设计题

### 73. 设计一个短链接系统，Redis 怎么用？

核心功能：

- 创建短链接。
- 根据短码跳转。
- 查询详情。
- 更新短链接。
- 查询访问统计。

数据库保存：

- code。
- original_url。
- user_id。
- status。
- expires_at。
- created_at。

Redis key：

```text
shortlink:detail:{code}
shortlink:visit_count:{code}
shortlink:create_limit:{user_id}
shortlink:not_found:{code}
```

Redis 用途：

- 缓存短链接详情。
- 记录访问次数。
- 用户创建限流。
- 空值缓存防止穿透。
- 更新后删除缓存。

跳转流程：

```text
读详情缓存
命中 -> 判断状态 -> INCR 计数 -> 302
未命中 -> 查数据库
数据库存在 -> 写缓存 -> INCR 计数 -> 302
数据库不存在 -> 写空值缓存 -> 404
```

更新流程：

```text
更新数据库
删除 shortlink:detail:{code}
删除 shortlink:not_found:{code}
```

降级：

- 缓存读失败查数据库。
- 缓存写失败记录日志。
- 计数失败不影响跳转。
- 创建限流失败保守拒绝。

---

### 74. 设计一个秒杀系统，Redis 怎么用？

Redis 可用于：

- 活动商品缓存。
- 库存预扣。
- 用户限流。
- 防重复下单。
- 请求排队。
- 热点数据缓存。

关键点：

- 库存扣减必须原子。
- 可以用 Lua 判断库存和用户是否已下单。
- 最终订单必须落数据库。
- Redis 预扣成功不等于订单最终成功。
- 要有异步下单和补偿。

Lua 思路：

```text
判断库存是否大于 0
判断用户是否已购买
扣减库存
记录用户已购买
写入订单消息
```

风险：

- Redis 数据和数据库数据一致性。
- 超卖。
- 重复下单。
- 热点 key。
- 消息积压。
- 恶意请求。

---

### 75. 设计一个排行榜系统，Redis 怎么用？

使用 ZSet。

key：

```text
rank:daily:20260705
rank:weekly:2026W27
rank:total
```

更新分数：

```redis
ZINCRBY rank:daily:20260705 10 user1001
```

查询 Top N：

```redis
ZREVRANGE rank:daily:20260705 0 99 WITHSCORES
```

查询个人排名：

```redis
ZREVRANK rank:daily:20260705 user1001
```

注意：

- 日榜设置 TTL。
- 总榜可能需要持久化到数据库。
- 分数更新频繁时注意热点。
- 大榜单分页不要一次取太多。

---

### 76. 设计一个验证码系统，Redis 怎么用？

Redis key：

```text
captcha:{phone}
captcha:send_limit:{phone}
captcha:verify_limit:{phone}
```

用途：

- 保存验证码，TTL 5 分钟。
- 发送频率限制，比如 60 秒一次。
- 验证次数限制，比如 5 次。

流程：

```text
发送验证码前检查发送限流
生成验证码
写入 Redis 并设置 TTL
验证时读取验证码
比对成功后删除
失败次数增加
超过次数拒绝
```

安全注意：

- 不要在日志打印验证码。
- 验证成功后删除验证码。
- 限制发送和验证频率。
- 防止短信轰炸。

---

### 77. 设计一个登录 token 系统，Redis 怎么用？

key：

```text
login:token:{token}
user:tokens:{user_id}
```

Redis 可保存：

- token 对应 user_id。
- 设备信息。
- 登录时间。
- 过期时间。

注意：

- token 设置 TTL。
- 登出时删除 token。
- 修改密码后清理用户所有 token。
- 多端登录要设计 token 集合。

如果 Redis 故障：

- 可能导致登录态不可用。
- 可以要求用户重新登录。
- 关键系统要考虑降级和高可用。

---

## 十五、开放题与追问

### 78. Redis 能保证强一致性吗？

Redis 本身不是强一致分布式数据库。

原因：

- 主从复制异步。
- 故障切换可能丢数据。
- 缓存和数据库之间天然存在一致性窗口。
- Redis Cluster 分片下多 key 操作受限制。

如果需要强一致：

- 使用数据库事务。
- 使用唯一约束。
- 使用幂等设计。
- 使用 etcd / ZooKeeper 等协调系统。
- Redis 只作为辅助加速或保护层。

---

### 79. Redis 宕机后如何恢复？

看部署方式：

- 单机：从 RDB/AOF 恢复。
- 主从 + Sentinel：自动故障转移。
- Cluster：从节点提升为主节点。

业务层：

- 缓存类数据可以回源数据库重建。
- 计数类数据可能丢失，需要看是否有落库。
- 限流类状态可以接受丢失或保守拒绝。
- 锁类状态必须谨慎处理。

恢复后要观察：

- 错误率。
- 命中率。
- 数据库 QPS。
- Redis 内存。
- 主从同步状态。

---

### 80. Redis 和数据库如何保持一致？

常见做法：

```text
读：Cache-Aside
写：更新数据库 -> 删除缓存
```

增强：

- 删除失败重试。
- 延迟双删。
- 消息队列补偿。
- binlog / CDC 订阅。
- TTL 兜底。

回答重点：

大多数缓存系统追求最终一致，不追求绝对强一致。要根据业务选择一致性策略。

---

### 81. Redis 为什么不建议使用 KEYS？

`KEYS pattern` 会遍历整个 keyspace。

问题：

- 时间复杂度高。
- 可能阻塞 Redis 主线程。
- key 很多时影响线上请求。

替代：

```redis
SCAN cursor MATCH pattern COUNT 100
```

SCAN 是渐进式遍历，不会一次阻塞太久。

但 SCAN 也不是完全无成本，生产使用仍要控制频率。

---

### 82. Redis 的删除为什么可能阻塞？

删除大 key 时，Redis 要释放大量内存结构。

如果一个 Hash、Set、ZSet、List 很大，`DEL` 可能耗时较长，阻塞主线程。

解决：

```redis
UNLINK key
```

或者分批删除集合元素。

---

### 83. Redis 中为什么要避免 value 太大？

大 value 会导致：

- 网络传输慢。
- 占用内存大。
- 序列化慢。
- 删除慢。
- 复制和持久化成本高。
- 客户端超时。

缓存对象应该尽量保存必要字段，不要把完整复杂对象无脑塞进 Redis。

---

### 84. Redis 中 JSON 缓存和 Hash 缓存怎么选？

JSON：

- 适合整体读写。
- 结构直观。
- 应用层反序列化方便。
- 更新单个字段需要重写整个 value。

Hash：

- 适合字段级读写。
- 可以单独更新字段。
- field 太多会变大 key。
- 嵌套结构不方便。

选择：

如果对象整体读写为主，用 JSON。

如果字段频繁单独更新，用 Hash。

---

### 85. Redis 中如何实现防重复提交？

可以用 `SET NX EX`。

示例：

```redis
SET submit:order:{request_id} 1 NX EX 60
```

如果设置成功，允许处理。

如果设置失败，说明重复提交。

注意：

Redis 防重复只是第一层，核心业务还要依赖数据库唯一约束或幂等表兜底。

---

### 86. Redis 可以做消息队列吗？

可以，但要看要求。

简单队列：

- List + `LPUSH` / `BRPOP`。

可靠一点：

- Stream + Consumer Group。

但如果需要：

- 高吞吐。
- 多分区。
- 完善重试。
- 复杂路由。
- 长期消息存储。
- 生态完善。

Kafka、RabbitMQ、RocketMQ 等专业消息队列更合适。

---

### 87. Redis Stream 和 Kafka 有什么区别？

Redis Stream：

- 使用简单。
- 适合轻量消息流。
- 与 Redis 生态结合方便。
- 容量和吞吐受 Redis 内存和单线程模型影响。

Kafka：

- 专业分布式日志系统。
- 高吞吐。
- 持久化强。
- 分区扩展能力强。
- 适合大规模事件流。

总结：

轻量异步事件可以用 Stream，大规模日志和事件平台更适合 Kafka。

---

### 88. Redis 中如何做多级缓存？

常见多级缓存：

```text
本地缓存 -> Redis -> 数据库
```

优点：

- 降低 Redis 压力。
- 提升极热数据访问速度。

难点：

- 本地缓存失效。
- 多实例一致性。
- 更新广播。
- 内存控制。

失效方式：

- 短 TTL。
- Redis Pub/Sub 广播删除。
- 消息队列通知。
- 配置中心推送。

适合：

- 热点配置。
- 热门商品详情。
- 允许短时间不一致的数据。

---

### 89. Redis 如何防止缓存污染？

缓存污染是指大量低价值数据进入缓存，挤掉真正热点数据。

解决：

- 只缓存访问频率达到阈值的数据。
- 对低频数据设置短 TTL。
- 使用 TinyLFU / 本地缓存策略。
- 限制缓存对象大小。
- 对异常请求做限流。
- 使用合适淘汰策略，如 LFU。

---

### 90. Redis 如何处理数据库回源压力？

手段：

- 缓存预热。
- 缓存 TTL 抖动。
- 空值缓存。
- 布隆过滤器。
- 互斥锁回源。
- SingleFlight 请求合并。
- 限流。
- 熔断。
- 降级。
- 多级缓存。

答题重点：

缓存系统的关键不是“缓存命中时快”，而是“缓存失效或 Redis 异常时不把数据库打爆”。

---

## 十六、面试场景快问快答

### 91. `SETNX` 和 `SET key value NX EX` 有什么区别？

`SETNX` 只负责 key 不存在时设置值，不能同时设置过期时间。

如果用：

```redis
SETNX lock value
EXPIRE lock 10
```

两条命令之间客户端宕机，可能导致锁没有过期时间。

推荐：

```redis
SET lock value NX EX 10
```

一条命令原子完成。

---

### 92. `DEL` 和 `UNLINK` 有什么区别？

`DEL` 同步删除 key 并释放内存，删除大 key 可能阻塞。

`UNLINK` 先把 key 从 keyspace 移除，内存释放交给后台线程异步处理。

删除大 key 推荐 `UNLINK`。

---

### 93. `EXPIRE` 和 `SET EX` 有什么区别？

`EXPIRE` 是给已有 key 设置过期时间。

`SET key value EX seconds` 是设置值时同时设置过期时间。

如果创建缓存，推荐直接：

```redis
SET key value EX 60
```

避免设置值和设置过期时间分离导致异常。

---

### 94. `SCAN` 会不会漏数据或重复数据？

SCAN 是渐进式遍历。

在遍历过程中，如果 keyspace 发生变化，可能出现重复，也可能看不到某些变化中的 key。

所以使用 SCAN 时：

- 要能处理重复。
- 不要依赖它做强一致快照。
- 适合渐进清理、排查、后台任务。

---

### 95. Redis 中 `INCR` 一个不存在的 key 会怎样？

如果 key 不存在，Redis 会把它当作 0，然后加 1。

结果为 1。

如果 key 存在但 value 不是整数，会报错。

---

### 96. Redis key 过期后一定会立刻删除吗？

不一定。

Redis 使用惰性删除和定期删除结合。

key 到期后，从逻辑上已经不可访问，但物理内存释放可能稍后发生。

---

### 97. Redis 主从复制是同步还是异步？

通常是异步复制。

主节点写成功后，不会等待所有从节点确认。

所以主节点故障时，可能丢失尚未复制到从节点的数据。

---

### 98. Redis Cluster 支持事务吗？

Redis Cluster 中事务要求所有 key 在同一个槽位。

如果多个 key 分散在不同槽，事务无法直接跨槽执行。

可以使用 Hash Tag 让相关 key 落到同一槽，但不能滥用。

---

### 99. Redis 里保存对象用一个大 JSON 好吗？

看场景。

如果对象较小，并且整体读写，JSON 可以。

如果对象很大或字段频繁单独更新，可能不适合。

风险：

- 每次更新要重写整个对象。
- value 太大导致传输和反序列化成本高。
- 删除可能变慢。

---

### 100. Redis 面试中最容易踩的坑是什么？

常见坑：

- 说 Redis 是单线程，但不知道后台线程和 I/O 多线程。
- 只会说缓存穿透、击穿、雪崩定义，不会讲项目处理。
- 分布式锁只说 `SETNX`，不说过期时间和 Lua 解锁。
- 说 Pipeline 是事务。
- 不知道 Redis Cluster 多 key 限制。
- 不知道 `KEYS` 生产危险。
- 不知道 `redis.Nil` 不是错误。
- 不知道缓存删除失败怎么补偿。
- 不知道 Redis 故障时业务如何降级。

---

## 十七、结合短链接项目的高频追问

### 101. 短链接跳转接口为什么不能每次查数据库？

因为跳转是高频读接口。

热门短链接可能被大量访问，如果每次都查数据库：

- 延迟高。
- 数据库 QPS 高。
- 高峰时容易打爆数据库。

Redis 缓存短链接详情后，大部分请求可以直接从 Redis 获取原始 URL。

---

### 102. 短链接不存在时为什么要做空值缓存？

因为攻击者可能随机请求大量不存在的短码。

如果不存在的短码每次都查数据库，会造成缓存穿透。

空值缓存可以把同一个不存在短码短时间挡在 Redis。

key：

```text
shortlink:not_found:{code}
```

TTL 要短，比如 30 到 60 秒。

---

### 103. 短链接访问计数为什么不应该影响跳转？

跳转是核心链路，计数是辅助统计。

如果 Redis 计数失败就拒绝跳转，会影响用户体验。

更合理：

- 计数成功最好。
- 计数失败记录日志。
- 继续跳转。

如果计数很重要，可以用 Stream 记录访问事件异步聚合。

---

### 104. 短链接更新后为什么删除缓存？

因为原始 URL、状态、过期时间变化后，旧缓存可能导致错误跳转。

更新流程：

```text
更新数据库
删除 shortlink:detail:{code}
删除 shortlink:not_found:{code}
```

下一次读取时重新从数据库加载最新数据。

---

### 105. 短链接创建接口 Redis 限流失败时怎么办？

创建短链接是写接口，可能被滥用。

如果 Redis 限流不可用，建议保守拒绝或进入降级保护，而不是直接放行。

原因：

- 放行可能导致垃圾数据大量写入。
- 数据库和短码空间会被滥用。

但具体策略要看业务重要性。

---

## 十八、面试答题模板

### 106. 回答 Redis 场景题的模板

可以按这个结构回答：

```text
1. 业务目标是什么？
2. 哪些数据存数据库，哪些数据存 Redis？
3. Redis key 如何设计？
4. TTL 如何设计？
5. 读流程是什么？
6. 写流程是什么？
7. 一致性如何保证？
8. Redis 故障如何降级？
9. 如何监控和排查？
10. 有什么边界和风险？
```

这样回答会比单纯背命令更像真实工程经验。

---

### 107. 回答缓存一致性问题的模板

```text
我们采用 Cache-Aside。
读请求先读缓存，未命中查数据库并回写缓存。
写请求先更新数据库，再删除缓存。
删除失败会记录日志并重试，必要时用消息队列或 binlog 订阅补偿。
缓存设置 TTL 作为兜底。
这个方案是最终一致，不是强一致。
如果业务强一致要求很高，不能只依赖缓存方案，要以数据库事务和约束为准。
```

---

### 108. 回答 Redis 故障降级问题的模板

```text
先区分 Redis 在这个业务里承担什么角色。
如果是详情缓存，Redis 故障可以回源数据库。
如果是计数器，可以短时间丢弃或异步补偿。
如果是排行榜，可以返回旧数据或空数据。
如果是限流、锁、验证码这类保护能力，故障时要更保守，可能拒绝请求。
同时要有日志、告警和监控，观察数据库 QPS 是否被回源打高。
```

---

### 109. 回答性能排查问题的模板

```text
我会先看现象范围：是所有接口慢，还是某个接口慢。
然后看应用日志和 Redis 客户端耗时，确认是 Redis 等待还是业务处理慢。
再看 Redis INFO、SLOWLOG、连接数、内存、命中率、淘汰 key。
如果 SLOWLOG 有慢命令，检查大 key、Lua、批量操作。
如果 SLOWLOG 没有慢命令，检查网络、连接池、客户端排队、Redis CPU 和数据库回源。
最后结合最近发布、流量变化、配置变更定位原因。
```

---

## 十九、最后复习清单

面试前至少确认自己能讲清楚：

1. Redis 为什么快。
2. Redis 单线程模型。
3. String、Hash、List、Set、ZSet 的使用场景。
4. Bitmap、HyperLogLog、Stream 的适用场景。
5. key 设计和 TTL 设计。
6. 缓存穿透、击穿、雪崩。
7. Cache-Aside。
8. 更新数据库后删除缓存。
9. 删除缓存失败补偿。
10. 分布式锁正确写法。
11. Lua 的作用和风险。
12. 固定窗口和滑动窗口限流。
13. Pipeline 和事务区别。
14. RDB 和 AOF 区别。
15. 内存淘汰策略。
16. big key 和 hot key 治理。
17. 主从、哨兵、Cluster。
18. Cluster 多 key 限制和 Hash Tag。
19. Redis 性能排查命令。
20. Redis 安全配置和监控指标。
21. Go 接入 Redis 的连接池、超时和 `redis.Nil`。
22. Redis 故障时业务降级。
23. 短链接项目 Redis 设计。

如果这些问题都能结合项目讲出来，Redis 面试基本就有底气了。

