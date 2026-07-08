# 8. 多 key 限制、hash tag 与综合实践

Redis Cluster 最容易踩坑的是多 key 操作。

在单实例 Redis 中，`MGET`、`MSET`、Lua 多 key 操作很自然。但在 Cluster 中，多 key 必须落在同一个 hash slot。

这一节把第 10 阶段串起来，并重点学习 hash tag。

学完这一节后，你应该能够：

- 理解 Cluster 多 key 限制。
- 使用 hash tag 控制 key 落到同一个 slot。
- 设计 Cluster 友好的 key。
- 搭建简单主从、Sentinel 和 Cluster 实践思路。
- 知道高可用架构下 Go 客户端要注意什么。

---

## 一、Cluster 中的多 key 限制

在 Cluster 中，下面命令要求所有 key 在同一个 slot：

```redis
MGET key1 key2
MSET key1 v1 key2 v2
DEL key1 key2
```

Lua 脚本如果访问多个 key，也要求同槽。

否则可能报错：

```text
CROSSSLOT Keys in request don't hash to the same slot
```

---

## 二、为什么有这个限制

Cluster 中不同 slot 可能在不同 master。

例如：

```text
key1 -> slot 100 -> master-1
key2 -> slot 9000 -> master-2
```

一个 Redis 节点无法原子操作另一个节点上的 key。

所以 Redis 要求多 key 在同一个 slot，确保它们在同一个 master 上执行。

---

## 三、hash tag 是什么

hash tag 是 key 中 `{}` 包裹的部分。

Redis Cluster 计算 slot 时，只使用 `{}` 内的内容。

例如：

```text
{user:1001}:profile
{user:1001}:settings
{user:1001}:tokens
```

这三个 key 都使用：

```text
user:1001
```

计算 slot，所以会落到同一个 slot。

---

## 四、hash tag 示例

查看：

```redis
CLUSTER KEYSLOT {user:1001}:profile
CLUSTER KEYSLOT {user:1001}:settings
```

它们应该返回相同 slot。

没有 hash tag：

```redis
CLUSTER KEYSLOT user:1001:profile
CLUSTER KEYSLOT user:1001:settings
```

不一定相同。

---

## 五、什么时候使用 hash tag

适合：

- 同一个用户多个 key 需要 `MGET`。
- Lua 脚本需要操作同一业务对象多个 key。
- 分布式锁和资源状态需要同槽。
- 订单相关多个 key 需要原子处理。

示例：

```text
order:{1001}:lock
order:{1001}:state
order:{1001}:token
```

---

## 六、不要滥用 hash tag

如果所有 key 都写成：

```text
{global}:key1
{global}:key2
{global}:key3
```

它们都会落到同一个 slot。

这会导致：

- 数据倾斜。
- 单个 master 压力过高。
- Cluster 分片失去意义。

hash tag 应该围绕业务对象使用。

例如用户、订单、租户，而不是全局常量。

---

## 七、Cluster 友好的 key 设计

单实例时：

```text
user:profile:1001
user:settings:1001
```

Cluster 中如果需要同槽，可以改成：

```text
user:{1001}:profile
user:{1001}:settings
```

订单：

```text
order:{1001}:detail
order:{1001}:status
order:{1001}:lock
```

短链接：

```text
shortlink:{abc123}:detail
shortlink:{abc123}:stats
```

---

## 八、Go 中多 key 操作

ClusterClient：

```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{"redis-7000:7000", "redis-7001:7001", "redis-7002:7002"},
})
```

同槽 MGET：

```go
keys := []string{
    "user:{1001}:profile",
    "user:{1001}:settings",
}

vals, err := rdb.MGet(ctx, keys...).Result()
```

跨槽 MGET 会失败。

所以 key 设计要提前考虑。

---

## 九、Lua 脚本同槽要求

例如脚本访问：

```text
lock:{order:1001}
state:{order:1001}
```

可以同槽。

如果写成：

```text
lock:order:1001
state:order:1001
```

不一定同槽。

Cluster 下 Lua 多 key 原子逻辑必须提前设计 hash tag。

---

## 十、综合实践 1：一主一从

实践目标：

```text
用 Docker Compose 启动 redis-master 和 redis-replica。
```

验证：

```redis
SET name redis
GET name
INFO replication
```

停止 master：

```bash
docker stop redis-master
```

观察：

```text
replica 不会自动变 master。
```

---

## 十一、综合实践 2：Sentinel

实践目标：

```text
一主一从 + 3 个 Sentinel。
```

验证：

```redis
SENTINEL masters
SENTINEL get-master-addr-by-name mymaster
```

停止 master 后观察：

- Sentinel 判断下线。
- replica 被提升。
- Go FailoverClient 自动找到新 master。

记录故障期间：

- 写失败次数。
- 恢复耗时。
- 是否有数据丢失。

---

## 十二、综合实践 3：Cluster

实践目标：

```text
启动 3 master + 3 replica Redis Cluster。
```

验证：

```redis
CLUSTER INFO
CLUSTER NODES
CLUSTER KEYSLOT user:1001
```

测试：

```redis
MGET user:1001:profile user:1001:settings
```

可能跨槽失败。

再测试：

```redis
MGET user:{1001}:profile user:{1001}:settings
```

同槽后可以执行。

---

## 十三、Go 客户端实践建议

三种客户端：

```go
// 单实例
redis.NewClient(...)

// Sentinel
redis.NewFailoverClient(...)

// Cluster
redis.NewClusterClient(...)
```

所有客户端都要考虑：

- `DialTimeout`
- `ReadTimeout`
- `WriteTimeout`
- `PoolSize`
- 重试次数
- 降级策略
- 日志和指标

部署形态变复杂后，客户端配置也必须认真。

---

## 十四、故障演练清单

建议演练：

- 停止 master。
- 停止 replica。
- 停止一个 Sentinel。
- 停止多个 Sentinel。
- Cluster 停止一个 master。
- Cluster 停止 master 和它的 replica。
- 网络延迟或断连。
- 客户端连接池耗尽。

每次记录：

- 故障发现时间。
- 切换耗时。
- 请求失败数量。
- 是否丢数据。
- 客户端是否自动恢复。

---

## 十五、常见错误

### 1. Cluster key 没考虑 hash tag

上线后多 key 命令大量 CROSSSLOT。

### 2. hash tag 使用全局常量

所有 key 落到同一个 slot，造成数据倾斜。

### 3. 没做故障演练

以为高可用能工作，真正故障时才发现客户端配置错。

### 4. 故障转移期间没有降级

短暂不可用也可能拖垮业务。

### 5. Redis 架构混用但代码无感

单实例、Sentinel、Cluster 的客户端和限制不同。

---

## 十六、本节练习

请完成下面练习：

1. 用 `CLUSTER KEYSLOT` 比较普通 key 和 hash tag key。
2. 设计用户相关的同槽 key。
3. 设计订单相关的同槽 key。
4. 在 Cluster 中测试跨槽 `MGET`。
5. 在 Cluster 中测试同槽 `MGET`。
6. 搭建一主一从并观察复制。
7. 搭建 Sentinel 并模拟 master 故障。
8. 用 Go 分别创建 FailoverClient 和 ClusterClient。
9. 记录一次故障演练结果。

---

## 十七、本节小结

这一节完成了第 10 阶段的实践闭环。

你需要记住：

- Cluster 中多 key 操作要求同一个 hash slot。
- hash tag 使用 `{}` 控制参与 slot 计算的部分。
- hash tag 要围绕业务对象设计，不能滥用全局 tag。
- Sentinel 解决主从自动故障转移。
- Cluster 解决分片扩展和高可用。
- 高可用仍然需要客户端超时、重试、降级、幂等和故障演练。

学完第 10 阶段，你已经理解 Redis 的主从复制、Sentinel 和 Cluster。下一阶段会进入性能分析与生产排查。

