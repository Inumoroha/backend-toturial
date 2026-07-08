# 08. 查询路由：targeted 与 scatter-gather

分片集群中，查询不是自动就快。mongos 会根据查询条件判断请求发到哪些 shard。是否包含 shard key，直接影响路由效率。

## 1. Targeted Query

如果查询包含 shard key 或 shard key 前缀，mongos 可以把查询定向到少数 shard。

例如 shard key：

```javascript
{ user_id: 1 }
```

查询：

```javascript
db.events.find({ user_id: userId })
```

mongos 可以定位到包含该 user_id 的 shard。

这叫 targeted query。

## 2. Scatter-Gather Query

如果查询不包含 shard key，mongos 可能把查询广播到所有 shard。

例如 shard key：

```javascript
{ user_id: 1 }
```

查询：

```javascript
db.events.find({ type: "click" })
```

mongos 不知道哪个 shard 有 `type: "click"`，可能发给所有 shard，再合并结果。

这叫 scatter-gather。

## 3. scatter-gather 的风险

风险：

- 所有 shard 都参与。
- 延迟取决于最慢 shard。
- 聚合和排序合并成本高。
- 高并发下影响整个集群。

后台低频分析可以接受，用户高频接口要尽量避免。

## 4. shard key 和查询模式

选择 shard key 时必须看高频查询。

日志事件核心查询：

```javascript
db.events.find({
  user_id: userId,
  created_at: { $gte: start, $lt: end }
})
```

如果 shard key 包含 `user_id`，查询更容易 targeted。

如果高频查询是：

```javascript
db.events.find({
  tenant_id: tenantId,
  created_at: { $gte: start, $lt: end }
})
```

则 shard key 要考虑 `tenant_id`。

## 5. 排序和 limit

分片查询中的排序和 limit 可能需要 mongos 合并多个 shard 结果。

如果 targeted 到一个 shard，成本较低。

如果 scatter 到所有 shard：

```javascript
db.events.find({ type: "click" }).sort({ created_at: -1 }).limit(20)
```

每个 shard 可能都返回候选结果，mongos 再合并排序。

## 6. 聚合中的路由

聚合 pipeline 也受影响。

好的做法：

```javascript
[
  { $match: { user_id: userId, created_at: { $gte: start, $lt: end } } },
  { $group: { ... } }
]
```

先用 shard key 过滤，减少参与 shard。

如果开头没有 shard key 条件，聚合可能跨所有 shard。

## 7. 查询设计建议

- 高频查询尽量包含 shard key。
- 后台分析可以接受 scatter，但要限流。
- 大范围聚合考虑离线系统。
- 列表接口要限制 `limit`。
- 避免无 shard key 的高频排序查询。

## 8. 本节练习

假设 shard key 是 `{ user_id: 1 }`，判断：

1. `{ user_id: u1 }`
2. `{ user_id: u1, created_at: { $gte: start } }`
3. `{ type: "click" }`
4. `{ created_at: { $gte: start, $lt: end } }`
5. `{ user_id: { $in: [u1, u2, u3] } }`

哪些是 targeted，哪些可能 scatter？

再假设 shard key 是 `{ tenant_id: 1, user_id: 1 }`，重复判断。

## 9. 本节小结

你需要记住：

- 包含 shard key 的查询更容易 targeted。
- 不包含 shard key 的查询可能 scatter-gather。
- scatter-gather 会放大查询成本。
- shard key 必须服务高频查询路径。
- 聚合也要尽量先用 shard key `$match`。

## 10. targeted query 的工程价值

targeted query 的核心价值是让 mongos 能把请求路由到少量 shard：

```text
请求量 1000 QPS
targeted 到 1 个 shard：约 1000 次 shard 查询
scatter 到 4 个 shard：约 4000 次 shard 查询
scatter 到 8 个 shard：约 8000 次 shard 查询
```

这就是为什么同样的业务 QPS，在分片集群里可能放大成数倍数据库压力。

## 11. 复合 shard key 的路由判断

假设 shard key：

```javascript
{ tenant_id: 1, user_id: 1 }
```

通常较好的查询：

```javascript
db.events.find({
  tenant_id: "t1",
  user_id: "u1",
  created_at: { $gte: start, $lt: end }
})
```

只带前缀字段：

```javascript
db.events.find({
  tenant_id: "t1",
  created_at: { $gte: start, $lt: end }
})
```

这种查询可能仍然比完全不带 shard key 好，因为它至少限定了租户范围。但如果一个租户跨很多 chunk 或 shard，仍可能访问多个 shard。

不好的查询：

```javascript
db.events.find({
  user_id: "u1"
})
```

它没有 shard key 前缀 `tenant_id`，路由效果通常会变差。复合 shard key 不是“包含任意一个字段就好”，字段顺序和前缀很重要。

## 12. 排序、分页与 scatter-gather

全局排序在分片集群中很昂贵。

```javascript
db.events.find({
  event_type: "click"
}).sort({ created_at: -1 }).limit(20)
```

如果没有 shard key 条件，mongos 可能需要：

```text
向多个 shard 发查询
每个 shard 返回一批排序结果
mongos 合并排序
取前 20 条
```

如果再加深分页：

```javascript
db.events.find({ event_type: "click" })
  .sort({ created_at: -1 })
  .skip(100000)
  .limit(20)
```

成本会更高。

更好的 API 设计：

```text
GET /tenants/{tenant_id}/events?cursor=...
GET /users/{user_id}/events?cursor=...
```

让查询天然带上路由维度，并使用游标分页。

## 13. 聚合管道中的路由

聚合也要尽量把 `$match` 放到前面，并包含 shard key：

```javascript
db.events.aggregate([
  {
    $match: {
      tenant_id: "t1",
      created_at: { $gte: start, $lt: end }
    }
  },
  {
    $group: {
      _id: "$event_type",
      count: { $sum: 1 }
    }
  }
])
```

不建议高频在线接口这样做：

```javascript
db.events.aggregate([
  { $group: { _id: "$event_type", count: { $sum: 1 } } }
])
```

这类全局聚合适合离线任务、数仓或预聚合表。

## 14. Go Repository 设计示例

不推荐：

```go
func (r *Repo) FindEvents(ctx context.Context, filter bson.M) ([]Event, error)
```

调用方可能传入任何条件，Repository 无法保证路由质量。

更推荐：

```go
func (r *Repo) ListTenantEvents(
    ctx context.Context,
    tenantID string,
    start time.Time,
    end time.Time,
    cursor time.Time,
    limit int64,
) ([]Event, error) {
    filter := bson.M{
        "tenant_id": tenantID,
        "created_at": bson.M{
            "$gte": start,
            "$lt":  end,
        },
    }
    if !cursor.IsZero() {
        filter["created_at"] = bson.M{
            "$gte": start,
            "$lt":  cursor,
        }
    }

    opts := options.Find().
        SetSort(bson.D{{Key: "created_at", Value: -1}}).
        SetLimit(limit)

    cur, err := r.events.Find(ctx, filter, opts)
    if err != nil {
        return nil, err
    }
    defer cur.Close(ctx)

    var out []Event
    if err := cur.All(ctx, &out); err != nil {
        return nil, err
    }
    return out, nil
}
```

这个方法强制调用方提供 `tenantID`，比任意 filter 更适合分片集群。

## 15. 慢查询排查问题清单

```text
这个查询是否包含 shard key？
是否包含 shard key 前缀？
是否命中合适索引？
是否有全局 sort？
是否有大 skip？
是否返回了大字段？
是否是在线接口里的全局聚合？
是否可以按租户/用户/时间桶拆分？
```

## 16. 官方文档延伸阅读

- [Targeted Operations vs. Broadcast Operations](https://www.mongodb.com/docs/manual/core/sharded-cluster-query-router/#targeted-operations-vs.-broadcast-operations)
- [Aggregation Pipeline and Sharded Collections](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-sharded-collections/)
