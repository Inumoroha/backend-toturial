# 05. chunk、balancer 与数据迁移

MongoDB 分片不是把每条文档单独随机放到 shard，而是按 shard key 把数据划分成 chunk，再把 chunk 分布到不同 shard。

## 1. 什么是 chunk

chunk 是 shard key 范围的一段数据。

例如 shard key 是：

```javascript
{ user_id: 1 }
```

chunk 可以理解为：

```text
user_id: MinKey -> A
user_id: A -> M
user_id: M -> Z
user_id: Z -> MaxKey
```

每个 chunk 存在某个 shard 上。

## 2. chunk 分裂

当某个 chunk 变大，MongoDB 可以把它拆成多个 chunk。

目的：

- 让数据可以更细粒度迁移。
- 避免单个 chunk 过大。

## 3. balancer

balancer 负责在 shard 之间迁移 chunk，让数据分布更均衡。

例如：

```text
shard1: 100 chunks
shard2: 30 chunks
shard3: 30 chunks
```

balancer 可能把一些 chunk 从 shard1 迁移到其他 shard。

## 4. chunk 迁移的成本

迁移会带来：

- 网络传输。
- 磁盘 IO。
- 复制压力。
- 对业务负载的影响。

所以生产中需要监控 balancer 活动和迁移窗口。

## 5. jumbo chunk

如果 chunk 太大且不能正常拆分，可能出现 jumbo chunk。

常见原因：

- shard key 选择差。
- 大量文档有相同 shard key 值。
- 低基数字段作为 shard key。

例如只用 `status` 做 shard key：

```text
paid / cancelled / pending
```

每个值下数据巨大，难以继续拆分。

## 6. 数据倾斜

即使 chunk 数量看起来均衡，也可能负载不均。

例如：

- 某个大租户访问量极高。
- 某个热门用户产生大量数据。
- 某个时间段写入集中。

所以要看：

- 数据量分布。
- QPS 分布。
- 写入分布。
- 热点 shard。

## 7. Zone Sharding 简介

MongoDB 支持 zone sharding，把特定范围数据放到指定 shard。

适合：

- 地域数据。
- 租户隔离。
- 冷热数据分层。

学习阶段了解即可。生产使用前要细读官方文档并充分测试。

## 8. 本节练习

回答：

1. chunk 是什么？
2. balancer 做什么？
3. chunk 迁移有什么成本？
4. 什么情况下可能出现 jumbo chunk？
5. 数据量均衡是否代表负载均衡？
6. 为什么低基数字段做 shard key 危险？

## 9. 本节小结

你需要记住：

- 分片数据以 chunk 为单位分布和迁移。
- balancer 负责平衡 chunk。
- chunk 迁移有成本。
- shard key 差会导致 jumbo chunk 和数据倾斜。
- 负载均衡不只看数据量，还要看读写热度。

## 10. chunk 的生命周期

可以把 chunk 理解成分片集群里的“数据范围单位”：

```text
创建集合分片
  -> 生成初始 chunk
  -> 数据写入
  -> chunk 变大
  -> chunk 分裂
  -> balancer 判断是否需要迁移
  -> chunk 从一个 shard 移到另一个 shard
```

chunk 不是业务文档，也不是集合。它是由 shard key 范围定义的数据区间。

例如 shard key 是：

```javascript
{ tenant_id: 1, created_at: 1 }
```

chunk 范围可能类似：

```text
["t001", MinKey] -> ["t001", 2026-07-01]
["t001", 2026-07-01] -> ["t002", MinKey]
["t002", MinKey] -> ["t010", MinKey]
```

真实输出会更复杂，但思想就是：每个 chunk 覆盖 shard key 空间中的一段。

## 11. 如何观察 chunk 和 balancer

在 mongosh 中查看分片状态：

```javascript
sh.status()
```

重点关注：

```text
集合是否 sharded
shard key 是什么
chunk 分布是否严重不均
是否有 jumbo 标记
是否配置了 zone
```

查看 balancer 状态：

```javascript
sh.getBalancerState()
```

如果需要观察迁移历史，可以查看 config 数据库中的相关集合。学习阶段先理解即可，生产中要结合监控平台观察迁移耗时、失败次数、迁移窗口和 shard 负载。

## 12. chunk 迁移为什么影响业务

chunk 迁移不是简单改一行元数据，它需要真实搬迁数据。迁移期间会涉及：

- 从源 shard 读取 chunk 数据。
- 复制到目标 shard。
- 追赶迁移期间的增量变更。
- 更新 config server 元数据。
- 清理源 shard 上的旧范围。

因此它会消耗：

- 网络带宽。
- 源 shard 和目标 shard 的磁盘 IO。
- CPU。
- oplog 和复制能力。
- config server 元数据写入能力。

如果业务高峰期大量迁移，用户请求延迟可能上升。

## 13. 数据均衡不等于负载均衡

看起来每个 shard 的 chunk 数量相同，不代表系统健康。

示例：

```text
shard1: 100 chunks，承载 1 个超级大租户
shard2: 100 chunks，承载 1000 个小租户
shard3: 100 chunks，承载 1000 个小租户
```

chunk 数量一样，但 shard1 的 QPS 可能远高于其他 shard。

所以要同时看：

```text
chunk 数量
数据大小
读 QPS
写 QPS
慢查询数量
磁盘 IO
CPU
复制延迟
```

## 14. jumbo chunk 的处理思路

出现 jumbo chunk 时，先不要急着执行危险操作。应该按顺序排查：

```text
1. shard key 是否低基数？
2. 是否大量文档拥有相同 shard key？
3. 是否存在超大租户或超大用户？
4. 是否可以细化 shard key？
5. 是否需要重新分片？
6. 是否能把冷数据归档？
```

常见根因：

```javascript
{ status: 1 }
```

`status` 只有几个值，单个值下面文档巨大，chunk 很难继续拆。

更好的方向：

```javascript
{ tenant_id: 1, status: 1, created_at: 1 }
```

或者基于业务改成：

```javascript
{ tenant_id: 1, bucket_month: 1, order_id: "hashed" }
```

## 15. 生产建议

- 不要在业务高峰期主动触发大量迁移。
- 分片前先估算初始数据分布。
- 大集合分片前要做压测。
- 监控 balancer 是否长时间迁移失败。
- 关注迁移期间复制延迟。
- 对大租户、大用户、大设备单独建监控。
- 把 chunk 分布和业务流量分布放在一起看。

## 16. 本节进阶练习

设计一个诊断报告：

```md
# 分片集群数据分布诊断

## 集合

## shard key

## chunk 分布

## 数据量分布

## QPS 分布

## 慢查询分布

## 是否存在 jumbo chunk

## 可能原因

## 调整建议
```

要求你分别给出“只看 chunk 数量”和“结合业务流量”时可能得出的不同结论。

## 17. 官方文档延伸阅读

- [Data Partitioning with Chunks](https://www.mongodb.com/docs/manual/core/sharding-data-partitioning/)
- [Sharded Cluster Balancer](https://www.mongodb.com/docs/manual/core/sharding-balancer-administration/)
