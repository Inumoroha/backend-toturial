# 01. 文档数据库与 BSON

学习 MongoDB 的第一步，是理解它和传统关系型数据库的思维差异。MongoDB 不是“没有表的 MySQL”，它的核心是文档模型。

## 1. 什么是文档数据库

文档数据库以“文档”为基本数据记录单位。

一个用户文档可以长这样：

```javascript
{
  _id: ObjectId("66a1f1c2e138237e2f7d0001"),
  name: "Alice",
  email: "alice@example.com",
  age: 25,
  roles: ["user", "editor"],
  profile: {
    city: "Shanghai",
    company: "Example Inc"
  },
  created_at: ISODate("2026-07-05T10:00:00Z")
}
```

这条文档包含：

- 字符串：`name`、`email`。
- 数字：`age`。
- 数组：`roles`。
- 嵌套文档：`profile`。
- 日期：`created_at`。
- ObjectId：`_id`。

这和关系型数据库一行一列的表格模型不同。MongoDB 的一条记录天然可以表达更丰富的结构。

## 2. MongoDB 和关系型数据库的类比

入门时可以先这样类比：

| 关系型数据库 | MongoDB |
| --- | --- |
| database | database |
| table | collection |
| row | document |
| column | field |
| primary key | `_id` |
| index | index |

但这个类比不能一直用下去。

关系型数据库通常先设计表，再通过 join 组合数据；MongoDB 更强调围绕查询场景设计文档结构。经常一起读取的数据，可以考虑放在同一个文档里。

## 3. 一个简单对比

### 关系型数据库中的订单

```text
orders
  id
  user_id
  status
  total_amount
  created_at

order_items
  id
  order_id
  product_id
  product_name
  price
  quantity
```

查询订单详情时，通常需要 join `orders` 和 `order_items`。

### MongoDB 中的订单

```javascript
{
  _id: ObjectId("66a1f1c2e138237e2f7d1001"),
  user_id: ObjectId("66a1f1c2e138237e2f7d0001"),
  status: "paid",
  total_amount: 19900,
  items: [
    {
      product_id: ObjectId("66a1f1c2e138237e2f7d2001"),
      product_name: "Go 后端实战",
      price: 9900,
      quantity: 1
    },
    {
      product_id: ObjectId("66a1f1c2e138237e2f7d2002"),
      product_name: "MongoDB 入门",
      price: 10000,
      quantity: 1
    }
  ],
  created_at: ISODate("2026-07-05T10:00:00Z")
}
```

订单明细被嵌入到订单文档中。查询订单详情时，一次读取就能拿到主要信息。

注意：这不是说 MongoDB 永远要嵌入。是否嵌入，要看数据是否总是一起读取、是否会无限增长、是否需要独立更新。

## 4. JSON 和 BSON

你看到的 MongoDB 文档很像 JSON，但 MongoDB 内部实际使用 BSON。

JSON 示例：

```json
{
  "name": "Alice",
  "age": 25,
  "created_at": "2026-07-05T10:00:00Z"
}
```

BSON 是 Binary JSON，可以理解为一种二进制编码格式。它支持比 JSON 更多的数据类型，例如：

- ObjectId
- Date
- Decimal128
- Binary
- Int32
- Int64
- Timestamp

官方文档的说法是：MongoDB 将数据记录存储为 BSON 文档，BSON 是 JSON 文档的二进制表示，并包含更多数据类型。

## 5. 为什么 Go 后端要关心 BSON

因为 Go 程序和 MongoDB 交互时，不是简单传字符串 JSON，而是通过 MongoDB Go Driver 把 Go 结构体编码成 BSON。

例如后面会写：

```go
type User struct {
    ID    primitive.ObjectID `bson:"_id,omitempty" json:"id"`
    Name  string             `bson:"name" json:"name"`
    Email string             `bson:"email" json:"email"`
    Age   int                `bson:"age" json:"age"`
}
```

这里的 `bson` tag 就是告诉驱动：这个 Go 字段和 MongoDB 文档里的哪个字段对应。

如果 BSON tag 写错，数据库里可能出现错误字段，或者查询不到数据。

## 6. 文档结构的优势

文档模型的优势：

- 可以自然表示嵌套对象。
- 可以自然表示数组。
- 单个文档可以承载一个聚合根。
- 读取常用组合数据时不一定需要 join。
- 结构灵活，适合业务快速变化。

例如用户资料：

```javascript
{
  name: "Alice",
  profile: {
    avatar: "https://example.com/a.png",
    city: "Shanghai",
    bio: "Go backend developer"
  },
  settings: {
    language: "zh-CN",
    email_notification: true
  }
}
```

这些资料经常和用户一起读取，嵌套在文档里很自然。

## 7. 文档结构的风险

文档模型也有风险：

- 字段太自由，容易变成脏数据。
- 大数组可能无限增长。
- 冗余数据需要同步。
- 跨文档事务比单文档操作更重。
- 不合理的嵌入会导致文档过大。

错误示例：把热门文章的所有评论都塞进文章文档。

```javascript
{
  title: "热门文章",
  comments: [
    // 可能增长到几万条
  ]
}
```

如果评论会无限增长，应该拆成单独的 `comments` 集合。

## 8. MongoDB 适合什么场景

MongoDB 常见适合场景：

- 内容系统。
- 用户画像。
- 商品目录。
- 订单快照。
- 日志和事件数据。
- 配置数据。
- 结构经常变化的业务数据。

不代表 MongoDB 只适合这些，也不代表这些场景只能用 MongoDB。真实选型要看查询模式、数据规模、一致性要求、团队经验和运维能力。

## 9. 初学者常见误区

### 误区 1：MongoDB 不需要设计表结构

MongoDB 不要求预先固定所有字段，但依然需要设计文档结构。

Go 后端中，结构体、参数校验、索引、Schema Validation 都会让数据结构保持稳定。

### 误区 2：能嵌套就全部嵌套

嵌套适合经常一起读取、生命周期一致、不会无限增长的数据。

### 误区 3：MongoDB 不支持事务

MongoDB 支持事务。但很多场景可以通过单文档原子更新解决，不应该一上来就滥用事务。

### 误区 4：不用索引也很快

小数据量时很多查询都快。数据一大，没有索引的问题会立刻暴露。

## 10. 本节练习

请判断下面数据适合嵌入还是拆集合，并写出理由：

1. 用户的收货地址。
2. 文章的前 3 条精选评论。
3. 文章的全部评论。
4. 订单中的商品快照。
5. 商品的库存流水。
6. 用户的登录日志。

参考思路：

- 是否总是和主文档一起读取？
- 是否会无限增长？
- 是否需要独立查询？
- 是否需要独立更新？
- 是否有不同生命周期？

## 11. 本节小结

你现在需要记住：

- MongoDB 的基本记录单位是 document。
- 文档可以包含嵌套对象和数组。
- MongoDB 内部使用 BSON，不是纯 JSON。
- 文档模型很灵活，但仍然需要认真设计。
- Go 后端学习 MongoDB 时，必须关心 BSON 和结构体映射。

