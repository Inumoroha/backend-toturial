# 08. 分页优化：从 skip 到游标分页

`skip + limit` 很直观，但深分页会变慢。后端工程师必须知道什么时候该从页码分页切换到游标分页。

## 1. 普通分页

```javascript
const page = 1000
const pageSize = 20

db.orders.find({ user_id: user1._id })
  .sort({ created_at: -1 })
  .skip((page - 1) * pageSize)
  .limit(pageSize)
```

问题：页码越深，skip 越大。

MongoDB 仍然需要跳过前面大量结果，成本会上升。

## 2. 深分页为什么慢

第 10000 页，每页 20 条：

```text
skip = 199980
```

数据库要先走过前面 199980 条匹配结果，再返回 20 条。

即使用了索引，也要扫描和跳过大量索引项。

## 3. 游标分页思想

不用页码，而是用上一页最后一条记录作为游标。

第一页：

```javascript
db.orders.find({ user_id: user1._id })
  .sort({ created_at: -1, _id: -1 })
  .limit(20)
```

拿到最后一条：

```javascript
lastCreatedAt
lastId
```

下一页：

```javascript
db.orders.find({
  user_id: user1._id,
  $or: [
    { created_at: { $lt: lastCreatedAt } },
    { created_at: lastCreatedAt, _id: { $lt: lastId } }
  ]
})
.sort({ created_at: -1, _id: -1 })
.limit(20)
```

## 4. 为什么要加 _id

如果只用 `created_at`：

```javascript
created_at: { $lt: lastCreatedAt }
```

当多条数据创建时间相同，可能漏数据或重复数据。

所以常用：

```text
created_at + _id
```

作为稳定排序。

## 5. 游标分页索引

用户订单游标分页：

```javascript
db.orders.createIndex({ user_id: 1, created_at: -1, _id: -1 })
```

查询：

```javascript
db.orders.find({ user_id: user1._id })
  .sort({ created_at: -1, _id: -1 })
  .limit(20)
```

下一页查询也要尽量匹配这个索引。

## 6. 游标分页适合什么

适合：

- 信息流。
- 订单列表。
- 评论列表。
- 消息列表。
- 日志列表。
- 无限滚动。

不适合：

- 必须跳到任意页码的后台表格。
- 需要准确总页数的场景。

后台管理如果必须跳页，可以接受 `skip`，但要限制最大页数或提供更强筛选条件。

## 7. API 设计

第一页：

```text
GET /orders?limit=20
```

返回：

```json
{
  "items": [],
  "next_cursor": "created_at_and_id_encoded"
}
```

下一页：

```text
GET /orders?limit=20&cursor=created_at_and_id_encoded
```

cursor 可以编码：

```json
{
  "created_at": "2026-07-05T10:00:00Z",
  "id": "..."
}
```

再 base64。

## 8. 游标分页的限制

限制：

- 不方便跳到第 N 页。
- 排序字段要稳定。
- 游标要防篡改，可以签名。
- 查询条件变化后 cursor 不能复用。

但对于大多数用户侧列表，游标分页更适合。

## 9. 本节练习

完成：

1. 用 `skip + limit` 查询用户订单第 100 页。
2. 用 explain 看扫描情况。
3. 创建 `{ user_id: 1, created_at: -1, _id: -1 }`。
4. 查询第一页。
5. 取最后一条的 `created_at` 和 `_id`。
6. 写下一页游标查询。
7. 对比 skip 深分页和游标分页思路。

## 10. 本节小结

你需要记住：

- `skip + limit` 简单但深分页慢。
- 游标分页用上一页最后一条作为下一页条件。
- 排序要稳定，常用 `created_at + _id`。
- 游标分页适合用户侧列表和无限滚动。
- 后台任意跳页需要额外限制和筛选条件。

