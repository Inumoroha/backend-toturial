# 04. 复合索引与 ESR 原则

真实项目里，最重要的往往不是单字段索引，而是复合索引。复合索引的字段顺序会直接影响查询是否高效。

## 1. 什么是复合索引

多个字段组成一个索引：

```javascript
db.orders.createIndex({ user_id: 1, status: 1, created_at: -1 })
```

它适合这类查询：

```javascript
db.orders.find({
  user_id: userId,
  status: "paid"
}).sort({ created_at: -1 })
```

## 2. 字段顺序很重要

这两个索引不同：

```javascript
{ user_id: 1, status: 1, created_at: -1 }
{ status: 1, user_id: 1, created_at: -1 }
```

哪个更好取决于查询模式。

如果高频查询总是：

```javascript
{ user_id: userId, status: "paid" }
```

并按 `created_at` 排序，那么第一个通常更贴近这个查询。

## 3. ESR 原则

ESR 是复合索引字段顺序的常用指导：

```text
Equality -> Sort -> Range
```

也就是：

1. 等值匹配字段。
2. 排序字段。
3. 范围查询字段。

例子：

```javascript
db.orders.find({
  user_id: userId,
  status: "paid",
  total_amount: { $gte: 10000 }
}).sort({ created_at: -1 })
```

可考虑索引：

```javascript
db.orders.createIndex({
  user_id: 1,
  status: 1,
  created_at: -1,
  total_amount: 1
})
```

`user_id`、`status` 是 Equality。

`created_at` 是 Sort。

`total_amount` 是 Range。

## 4. Equality 字段顺序

多个等值字段之间，顺序通常没有排序和范围那么敏感，但仍要结合查询频率。

例如：

```javascript
{ user_id: userId, status: "paid" }
```

如果还有大量查询只按 `user_id`：

```javascript
db.orders.find({ user_id: userId })
```

那么：

```javascript
{ user_id: 1, status: 1, created_at: -1 }
```

比：

```javascript
{ status: 1, user_id: 1, created_at: -1 }
```

更能兼顾按用户查询。

## 5. 排序字段的位置

查询：

```javascript
db.orders.find({
  user_id: userId,
  status: "paid"
}).sort({ created_at: -1 })
```

索引：

```javascript
{ user_id: 1, status: 1, created_at: -1 }
```

可以支持过滤和排序。

如果索引没有 `created_at`：

```javascript
{ user_id: 1, status: 1 }
```

MongoDB 可能需要额外内存排序。

## 6. 范围查询字段

范围条件：

```javascript
created_at: { $gte: start, $lt: end }
total_amount: { $gte: 10000 }
age: { $gt: 18 }
```

一旦索引使用到范围字段，后续字段的可用性可能受限。

例子：

```javascript
db.orders.find({
  status: "paid",
  created_at: { $gte: start, $lt: end }
}).sort({ total_amount: -1 })
```

这里排序和范围字段的选择要结合实际 explain，不要只凭感觉。

## 7. 索引前缀

复合索引：

```javascript
{ user_id: 1, status: 1, created_at: -1 }
```

可以支持前缀：

```javascript
{ user_id: 1 }
{ user_id: 1, status: 1 }
{ user_id: 1, status: 1, created_at: -1 }
```

但不能很好支持只按：

```javascript
{ status: "paid" }
```

因为 `status` 不是索引前缀的第一个字段。

这就是字段顺序重要的原因。

## 8. 订单列表索引设计

用户订单列表：

```javascript
db.orders.find({ user_id: userId }).sort({ created_at: -1 }).limit(20)
```

索引：

```javascript
db.orders.createIndex({ user_id: 1, created_at: -1 })
```

用户按状态筛选订单：

```javascript
db.orders.find({
  user_id: userId,
  status: "paid"
}).sort({ created_at: -1 }).limit(20)
```

索引：

```javascript
db.orders.createIndex({ user_id: 1, status: 1, created_at: -1 })
```

后台按状态看订单：

```javascript
db.orders.find({ status: "paid" }).sort({ created_at: -1 }).limit(50)
```

索引：

```javascript
db.orders.createIndex({ status: 1, created_at: -1 })
```

不要以为一个索引能完美覆盖所有查询。

## 9. 索引方向

单字段排序：

```javascript
{ created_at: -1 }
```

可以支持倒序，也通常可以反向扫描支持升序。

复合排序方向更复杂。

例如：

```javascript
{ status: 1, created_at: -1 }
```

适合：

```javascript
sort({ status: 1, created_at: -1 })
```

或完全反向：

```javascript
sort({ status: -1, created_at: 1 })
```

不要随意混合方向，实际用 explain 验证。

## 10. 本节练习

为下面查询设计索引：

1. 查询某用户订单，按创建时间倒序。
2. 查询某用户某状态订单，按创建时间倒序。
3. 后台查询 paid 订单，按创建时间倒序。
4. 查询某时间范围内 paid 订单，按金额倒序。
5. 查询某城市 active 用户，按创建时间倒序。
6. 查询某文章评论，按创建时间倒序。

格式：

```text
查询：
Equality：
Sort：
Range：
建议索引：
理由：
```

## 11. 本节小结

你需要记住：

- 复合索引字段顺序非常重要。
- ESR 是 Equality、Sort、Range。
- 复合索引可以支持索引前缀。
- 一个索引不一定覆盖所有查询。
- 设计索引后必须用 explain 验证。

