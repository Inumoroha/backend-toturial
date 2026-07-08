# 7. Redis Cluster：Hash Slot、MOVED 与 ASK 重定向

Sentinel 解决主从架构的自动故障转移，但所有数据仍然在一个 master 上。

Redis Cluster 通过分片把数据分布到多个 master 上，解决单节点容量和吞吐限制。

学完这一节后，你应该能够：

- 理解 Redis Cluster 的基本结构。
- 理解 16384 个 hash slot。
- 知道 key 如何映射到 slot。
- 理解 `MOVED` 和 `ASK` 重定向。
- 使用 go-redis ClusterClient。

---

## 一、Cluster 是什么

Redis Cluster 是 Redis 官方分布式方案。

它通常由多个 master 和 replica 组成：

```text
master-1 + replica-1
master-2 + replica-2
master-3 + replica-3
```

数据被分配到不同 master。

每个 master 负责一部分 slot。

---

## 二、Hash Slot

Redis Cluster 固定有：

```text
16384 个 hash slot
```

每个 key 会映射到其中一个 slot。

然后根据 slot 找到负责它的 master。

例如：

```text
slot 0-5460      -> master-1
slot 5461-10922  -> master-2
slot 10923-16383 -> master-3
```

---

## 三、key 如何映射 slot

简化理解：

```text
slot = CRC16(key) % 16384
```

查看 key 的 slot：

```redis
CLUSTER KEYSLOT user:1001
```

不同 key 大概率落在不同 slot。

Cluster 客户端会根据 slot 把命令发到对应节点。

---

## 四、MOVED 重定向

如果客户端把命令发错节点，Redis 会返回：

```text
MOVED 3999 127.0.0.1:7001
```

含义：

```text
这个 slot 不在当前节点，
请去 127.0.0.1:7001。
```

Cluster 客户端收到 MOVED 后，会更新本地 slot 路由表。

go-redis ClusterClient 会自动处理。

---

## 五、ASK 重定向

`ASK` 通常发生在 slot 迁移过程中。

例如 slot 正在从 A 节点迁移到 B 节点。

Redis 可能返回：

```text
ASK 3999 127.0.0.1:7002
```

客户端需要临时去目标节点执行一次命令。

和 MOVED 不同：

- MOVED 表示 slot 已经归属新节点。
- ASK 表示迁移过程中的临时重定向。

Cluster 客户端通常会自动处理。

---

## 六、Cluster 节点之间会通信

Cluster 节点之间使用 gossip 协议交换信息。

它们会感知：

- 节点是否在线。
- slot 分布。
- 主从关系。
- 故障状态。

所以 Cluster 不是几个独立 Redis 简单拼在一起。

它们共同维护集群拓扑。

---

## 七、Cluster 高可用

每个 master 通常至少有一个 replica。

如果 master 挂了：

```text
它的 replica 可以被提升为新 master。
```

如果某个 master 没有 replica，且它负责的 slot 不可用，集群可能进入不可用状态。

所以生产 Cluster 需要合理副本。

---

## 八、go-redis ClusterClient

```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{
        "redis-7000:7000",
        "redis-7001:7001",
        "redis-7002:7002",
    },
    ReadTimeout:  200 * time.Millisecond,
    WriteTimeout: 200 * time.Millisecond,
    DialTimeout:  500 * time.Millisecond,
})
```

不需要列出所有节点，但建议提供多个入口节点。

客户端会发现集群拓扑。

---

## 九、Cluster 中读 replica

go-redis 支持从 replica 读：

```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs:           []string{"redis-7000:7000", "redis-7001:7001"},
    ReadOnly:        true,
    RouteByLatency:  true,
    RouteRandomly:   true,
})
```

但和主从读写分离一样，要考虑复制延迟。

强一致读不要随便读 replica。

---

## 十、Cluster 的好处

好处：

- 数据分片，突破单节点内存限制。
- 多 master 分担请求。
- 支持节点故障转移。
- 客户端自动路由到 slot 节点。

适合：

- 数据量较大。
- 单机 Redis 内存不够。
- 单节点吞吐不够。
- 需要官方分片方案。

---

## 十一、Cluster 的复杂度

复杂度：

- 多 key 命令受限制。
- Lua 脚本访问多个 key 要同槽。
- 客户端要支持 MOVED/ASK。
- slot 迁移需要谨慎。
- 运维复杂度更高。
- 故障排查比单实例和 Sentinel 更复杂。

如果单实例或 Sentinel 已经够用，不要为了“高级”上 Cluster。

---

## 十二、常见错误

### 1. 使用普通 Client 连接 Cluster

普通客户端无法正确处理 slot 路由。

### 2. 以为 MOVED 是异常故障

MOVED 是 Cluster 正常重定向机制。

### 3. 多 key 随便跨 slot

Cluster 会拒绝跨 slot 多 key 操作。

### 4. Cluster 中读 replica 不考虑延迟

仍然可能读到旧数据。

### 5. 所有节点都放同一机器

机器故障时整个集群受影响。

---

## 十三、本节练习

请完成下面练习：

1. 用自己的话解释 hash slot。
2. 执行 `CLUSTER KEYSLOT user:1001`。
3. 解释 MOVED 和 ASK 的区别。
4. 用 go-redis 创建 ClusterClient。
5. 思考为什么建议配置多个入口节点。
6. 判断什么情况下需要 Redis Cluster。
7. 列出 Cluster 相比 Sentinel 的复杂点。

---

## 十四、本节小结

这一节你学习了 Redis Cluster 基础。

你需要记住：

- Redis Cluster 使用 16384 个 hash slot 分片。
- key 通过 CRC16 映射到 slot。
- MOVED 表示 slot 归属变化，ASK 表示迁移中的临时重定向。
- ClusterClient 会自动处理 slot 路由和重定向。
- Cluster 提供分片扩展，但也带来多 key 限制和运维复杂度。

下一节我们学习多 key 限制、hash tag 和综合实践。

