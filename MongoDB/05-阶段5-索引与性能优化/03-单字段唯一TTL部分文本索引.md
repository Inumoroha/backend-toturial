# 03. 单字段、唯一、TTL、部分、文本索引

本节学习常见索引类型。先掌握它们解决什么问题，再学复合索引字段顺序。

## 1. 单字段索引

给用户邮箱建索引：

```javascript
db.users.createIndex({ email: 1 })
```

查看：

```javascript
db.users.getIndexes()
```

查询：

```javascript
db.users.find({ email: "user100@example.com" })
```

`1` 表示升序，`-1` 表示降序。对于等值查询，升序或降序通常差别不大。

## 2. 唯一索引

邮箱不应该重复：

```javascript
db.users.createIndex({ email: 1 }, { unique: true })
```

如果已经存在重复 email，创建会失败。

唯一索引适合：

- 用户邮箱。
- 手机号。
- 订单号。
- 支付流水号。
- 幂等键。

唯一索引不仅提升查询性能，也保证业务约束。

## 3. 稀疏索引 sparse

如果有些用户没有 phone：

```javascript
db.users.createIndex({ phone: 1 }, { unique: true, sparse: true })
```

`sparse` 只索引包含该字段的文档。

注意：稀疏索引和部分索引都要谨慎理解，真实项目中更推荐明确使用 partial index 表达业务条件。

## 4. 部分索引 partial index

只给 active 用户的 email 建唯一索引：

```javascript
db.users.createIndex(
  { email: 1 },
  {
    unique: true,
    partialFilterExpression: {
      status: "active"
    }
  }
)
```

更常见：软删除后允许复用邮箱。

```javascript
db.users.createIndex(
  { email: 1 },
  {
    unique: true,
    partialFilterExpression: {
      deleted_at: { $exists: false }
    }
  }
)
```

部分索引只包含符合条件的文档。

适合：

- 只查询 active 数据。
- 软删除场景唯一约束。
- 大集合中只索引少量热数据。

## 5. TTL 索引

验证码、临时 token 需要自动过期：

```javascript
db.verify_tokens.createIndex(
  { expired_at: 1 },
  { expireAfterSeconds: 0 }
)
```

文档：

```javascript
{
  email: "user1@example.com",
  token: "token-1",
  expired_at: ISODate("2026-07-05T10:05:00Z")
}
```

当 `expired_at` 到期后，MongoDB 后台 TTL monitor 会自动删除过期文档。

注意：

- TTL 删除不是精确到秒实时删除。
- TTL 字段必须是日期类型或日期数组。
- 不要用 TTL 删除订单、支付流水、审计日志这类重要数据。

## 6. 文本索引 text index

给文章标题和内容建文本索引：

```javascript
db.posts.createIndex({
  title: "text",
  content: "text"
})
```

搜索：

```javascript
db.posts.find({
  $text: { $search: "MongoDB 索引" }
})
```

按文本得分排序：

```javascript
db.posts.find(
  { $text: { $search: "MongoDB 索引" } },
  { score: { $meta: "textScore" }, title: 1 }
).sort({
  score: { $meta: "textScore" }
})
```

文本索引适合简单全文搜索。复杂搜索、高亮、分词、相关性排序等需求可以考虑 Atlas Search 或 Elasticsearch。

## 7. 查看和删除索引

查看：

```javascript
db.users.getIndexes()
```

删除指定索引：

```javascript
db.users.dropIndex("email_1")
```

删除所有非 `_id` 索引：

```javascript
db.users.dropIndexes()
```

## 8. 索引命名

默认索引名：

```text
email_1
user_id_1_created_at_-1
```

可以自定义：

```javascript
db.orders.createIndex(
  { user_id: 1, created_at: -1 },
  { name: "idx_orders_user_created_at" }
)
```

生产项目建议给重要索引命名，方便沟通和迁移。

## 9. 本节练习

完成：

1. 给 `users.email` 建唯一索引。
2. 给 `orders.order_no` 建唯一索引。
3. 给 `verify_tokens.expired_at` 建 TTL 索引。
4. 给 `posts.title/content` 建文本索引。
5. 使用 `$text` 搜索文章。
6. 查看所有集合索引。
7. 删除一个索引再重建。

## 10. 本节小结

你需要记住：

- 单字段索引用于简单字段查询。
- 唯一索引用于保证业务唯一性。
- TTL 索引用于临时数据自动过期。
- 部分索引用于只索引符合条件的数据。
- 文本索引用于基础全文搜索。
- 索引要能解释业务目的，不要盲目创建。

