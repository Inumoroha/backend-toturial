# 03. shard key 设计原则

shard key 是分片集合最关键的设计。它决定数据如何分布、查询如何路由、写入是否热点、后续扩展是否顺畅。

## 1. 什么是 shard key

shard key 是分片集合中用于分布数据的字段或字段组合。

例如：

```javascript
{ user_id: 1 }
```

或：

```javascript
{ tenant_id: 1, created_at: 1 }
```

MongoDB 根据 shard key 把文档分配到不同 chunk，再把 chunk 分布到不同 shard。

## 2. shard key 为什么重要

它影响：

- 数据分布是否均匀。
- 写入是否集中到单个 shard。
- 查询是否能定向到少数 shard。
- chunk 是否容易迁移。
- 唯一索引如何设计。
- 后续扩展成本。

选错 shard key，后面很难补救。

## 3. 好 shard key 的特征

### 高基数

可选值很多。

好：

```text
user_id
device_id
tenant_id + event_time
```

差：

```text
status
gender
type
```

`status` 只有几个值，容易分布不均。

### 分布均匀

数据能分散到多个 shard。

如果大部分数据都属于同一个 tenant，大租户可能成为热点。

### 查询经常包含

查询如果包含 shard key，mongos 可以做 targeted query。

如果查询不包含 shard key，可能 scatter-gather 到所有 shard。

### 写入不集中

不要让所有新写入都集中到同一个 chunk 或 shard。

单调递增字段如纯 `created_at` 容易热点。

## 4. 常见候选 shard key

### user_id

适合按用户查询的业务。

优点：

- 高基数。
- 用户维度查询可定向。

风险：

- 超级用户可能形成热点。
- 按时间范围全局查询可能跨 shard。

### tenant_id

适合多租户 SaaS。

优点：

- 租户隔离清晰。
- 租户查询定向。

风险：

- 大租户倾斜。
- 小租户太多但访问不均。

### device_id

适合 IoT。

优点：

- 设备维度查询定向。
- 基数高。

风险：

- 热门设备高频写。
- 全局时间查询跨 shard。

### hashed user_id

适合打散写入。

优点：

- 分布更均匀。

风险：

- 范围查询能力弱。
- 按 user_id 定向可用，但按范围不友好。

## 5. 复合 shard key

例子：

```javascript
{ tenant_id: 1, created_at: 1 }
```

适合：

- 大多数查询包含 tenant_id。
- 同一租户内按时间查询。

风险：

- 如果 tenant_id 倾斜，大租户仍可能热点。
- created_at 单调增长可能在某些模式下形成局部热点。

另一种：

```javascript
{ tenant_id: 1, user_id: 1 }
```

适合租户内按用户查询。

## 6. shard key 设计流程

```text
1. 列出核心查询
2. 列出写入模式
3. 估算字段基数
4. 评估数据分布
5. 评估是否有热点
6. 评估查询是否包含 shard key
7. 评估唯一索引需求
8. 设计候选 shard key
9. 用样本数据和压测验证
```

## 7. 不要选这些字段

通常不适合单独作为 shard key：

- `created_at`
- 自增 ID
- `status`
- `type`
- `is_deleted`
- `gender`
- 低基数字段

原因：

- 要么热点。
- 要么分布差。
- 要么查询路由效果差。

## 8. 本节练习

为下面集合设计候选 shard key：

1. 订单集合。
2. 评论集合。
3. 日志事件集合。
4. IoT 设备指标集合。
5. 多租户 SaaS 的工单集合。
6. 用户消息集合。

格式：

```text
集合：
核心查询：
写入模式：
候选 shard key：
优点：
风险：
是否需要验证：
```

## 9. 本节小结

你需要记住：

- shard key 是分片最关键设计。
- 好 shard key 要高基数、分布均匀、查询常包含、不产生写热点。
- 低基数字段不适合单独作为 shard key。
- 单调递增字段容易热点。
- shard key 要基于查询和写入模式设计。

## 10. shard key 评估打分表

选择 shard key 时，可以给每个候选字段打分：

```text
字段组合：

基数：        1-5 分
写入离散度：  1-5 分
查询命中率：  1-5 分
是否避免热点：1-5 分
是否支持排序：1-5 分
是否稳定：    1-5 分
是否易理解：  1-5 分
```

示例：

| 候选 shard key | 基数 | 写入离散 | 查询命中 | 热点风险 | 说明 |
| --- | ---: | ---: | ---: | ---: | --- |
| `{ created_at: 1 }` | 高 | 低 | 中 | 高 | 时间递增，容易集中写入最后一个 chunk |
| `{ user_id: 1 }` | 高 | 中 | 高 | 中 | 适合按用户查询，但大用户可能热点 |
| `{ tenant_id: 1, created_at: 1 }` | 中/高 | 中 | 高 | 中 | 适合多租户按时间查询 |
| `{ tenant_id: "hashed" }` | 中/高 | 高 | 中 | 低 | 写入均匀，但范围查询能力下降 |

不要只看一个指标。能均匀写入但无法支持主要查询，也不是好 shard key。

## 11. 复合 shard key 的常见设计

### 多租户系统

常见查询：

```text
按 tenant_id 查询订单
按 tenant_id + created_at 查询订单列表
按 tenant_id + status 查询订单
```

候选：

```javascript
{ tenant_id: 1, created_at: 1 }
```

优点：

- 大多数租户查询可以 targeted。
- 支持租户内时间范围。
- 和业务权限边界一致。

风险：

- 超大租户可能形成热点。
- 如果所有写入都集中在少数租户，需要更细粒度字段。

### 用户行为事件

常见查询：

```text
按 app_id + 时间范围统计
按 user_id 查询最近行为
按 event_type 做离线分析
```

候选：

```javascript
{ app_id: 1, bucket_day: 1, user_id: "hashed" }
```

这里的 `bucket_day` 是人为构造的日期桶，例如 `2026-07-05`。它比直接用秒级时间更容易控制范围，也更容易归档。

## 12. shard key 与索引的关系

分片集合需要支持 shard key 的索引。实际设计时，不要只建 shard key 索引，还要建业务查询索引。

例如 shard key：

```javascript
{ tenant_id: 1, created_at: 1 }
```

常见查询：

```javascript
db.orders.find({
  tenant_id: "t1001",
  status: "paid",
  created_at: { $gte: ISODate("2026-07-01") }
}).sort({ created_at: -1 })
```

可能需要：

```javascript
db.orders.createIndex({
  tenant_id: 1,
  status: 1,
  created_at: -1
})
```

这类索引既帮助 mongos 定位 shard，也帮助 shard 内部高效扫描。

## 13. 什么时候需要重新评估 shard key

出现下面情况时，要重新评估：

- 新业务查询不再带原 shard key。
- 数据量均衡但某个 shard QPS 特别高。
- 出现 jumbo chunk。
- 大租户越来越大。
- 大量查询变成 scatter-gather。
- 写入集中在同一个时间范围或同一个业务键。

MongoDB 支持重新分片能力，但重新分片不是无成本操作。它会占用集群资源，也需要压测、监控和回滚方案。

## 14. 设计练习：选择 shard key

假设你要设计一个 SaaS 审计日志系统：

```text
租户数：3000
大租户：前 20 个贡献 70% 日志
写入：每天 2 亿条
查询：
  1. 租户管理员查最近 7 天日志
  2. 安全团队按 user_id 查行为
  3. 平台做离线统计
保留：180 天
```

请比较：

```javascript
{ tenant_id: 1, created_at: 1 }
{ tenant_id: "hashed" }
{ tenant_id: 1, bucket_day: 1, user_id: "hashed" }
```

回答：

- 哪个更适合在线查询？
- 哪个写入更均匀？
- 哪个更容易产生大租户热点？
- 哪个更适合归档？
- 需要哪些辅助索引？

## 15. 官方文档延伸阅读

- [Shard Keys](https://www.mongodb.com/docs/manual/core/sharding-shard-key/)
- [Choose a Shard Key](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/)
