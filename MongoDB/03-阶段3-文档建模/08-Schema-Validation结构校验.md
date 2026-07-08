# 08. Schema Validation 结构校验

MongoDB 是灵活 Schema，但灵活不等于没有约束。真实项目中，你应该用 Go 结构体、业务校验和 MongoDB Schema Validation 一起保证数据质量。

## 1. 为什么需要 Schema Validation

如果没有约束，同一个集合可能出现：

```javascript
{ email: "alice@example.com", age: 25 }
{ emial: "bob@example.com", age: "30" }
{ email: 123, status: "hello" }
```

这些脏数据会导致：

- Go Decode 失败。
- 查询条件失效。
- 索引无法按预期使用。
- 统计结果错误。
- 维护成本上升。

Schema Validation 可以在数据库层拒绝明显错误的数据。

## 2. MongoDB Validation 基本形式

创建集合时指定验证规则：

```javascript
db.createCollection("users_validated", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name", "status", "created_at", "updated_at"],
      properties: {
        email: {
          bsonType: "string",
          description: "email must be a string and is required"
        },
        name: {
          bsonType: "string"
        },
        age: {
          bsonType: "int",
          minimum: 0
        },
        status: {
          enum: ["active", "inactive", "blocked"]
        },
        roles: {
          bsonType: "array",
          items: {
            bsonType: "string"
          }
        },
        created_at: {
          bsonType: "date"
        },
        updated_at: {
          bsonType: "date"
        }
      }
    }
  }
})
```

官方文档说明，Validation 默认会拒绝违反规则的插入或更新。

## 3. 测试校验

正确插入：

```javascript
db.users_validated.insertOne({
  email: "alice@example.com",
  name: "Alice",
  age: 25,
  status: "active",
  roles: ["user"],
  created_at: new Date(),
  updated_at: new Date()
})
```

错误插入：

```javascript
db.users_validated.insertOne({
  email: 123,
  name: "Bob",
  status: "active",
  created_at: new Date(),
  updated_at: new Date()
})
```

会因为 `email` 类型错误被拒绝。

## 4. validationLevel

可以设置：

```javascript
validationLevel: "strict"
```

或：

```javascript
validationLevel: "moderate"
```

常见理解：

- `strict`：对所有插入和更新执行校验。
- `moderate`：对已有不符合规则的文档更宽松，适合逐步迁移。

学习阶段优先使用 `strict`。

## 5. validationAction

可以设置：

```javascript
validationAction: "error"
```

或：

```javascript
validationAction: "warn"
```

常见理解：

- `error`：违反规则时报错，拒绝写入。
- `warn`：记录警告，但允许写入。

学习阶段建议使用 `error`，这样能尽早发现问题。

## 6. 修改已有集合校验规则

```javascript
db.runCommand({
  collMod: "users_validated",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name", "status"],
      properties: {
        email: { bsonType: "string" },
        name: { bsonType: "string" },
        status: { enum: ["active", "inactive", "blocked"] }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})
```

真实项目中修改校验规则前要先检查历史数据，否则上线后可能导致写入失败。

## 7. Schema Validation 和 Go 校验的关系

不要只依赖数据库校验。

推荐多层校验：

```text
Handler 参数校验
  -> Service 业务校验
  -> Repository 写入
  -> MongoDB Schema Validation 兜底
```

例如邮箱格式，最好在 API 层先校验。数据库 Schema 可以保证它是字符串，但复杂格式校验不一定都适合放数据库。

## 8. 什么适合放 Schema Validation

适合：

- 必填字段。
- 基础类型。
- 枚举值。
- 数字范围。
- 数组元素类型。
- 子文档基本结构。

不适合全部放数据库：

- 复杂业务规则。
- 跨集合存在性校验。
- 权限判断。
- 需要调用外部服务的校验。
- 频繁变化的实验性字段。

## 9. 订单集合校验示例

```javascript
db.createCollection("orders_validated", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["order_no", "user_id", "status", "items", "total_amount", "created_at"],
      properties: {
        order_no: { bsonType: "string" },
        user_id: { bsonType: "objectId" },
        status: {
          enum: ["pending_payment", "paid", "shipped", "completed", "cancelled"]
        },
        items: {
          bsonType: "array",
          minItems: 1,
          items: {
            bsonType: "object",
            required: ["product_id", "product_name", "price", "quantity"],
            properties: {
              product_id: { bsonType: "objectId" },
              product_name: { bsonType: "string" },
              price: { bsonType: "long" },
              quantity: { bsonType: "int", minimum: 1 }
            }
          }
        },
        total_amount: { bsonType: "long", minimum: 0 },
        created_at: { bsonType: "date" },
        updated_at: { bsonType: "date" }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})
```

注意：金额字段建议用整数分，避免浮点误差。

## 10. 版本化 Schema

业务演进时，可以给文档加版本字段：

```javascript
{
  schema_version: 2,
  email: "alice@example.com",
  name: "Alice"
}
```

好处：

- 支持平滑迁移。
- 读取旧数据时可以兼容。
- 迁移脚本可以按版本处理。

如果系统变化频繁，Schema 版本是很实用的工程手段。

## 11. 本节练习

为 `products` 集合设计 Schema Validation，要求：

- `name` 必填，字符串。
- `category` 必填，字符串。
- `status` 只能是 `draft`、`on_sale`、`off_sale`。
- `price` 必填，整数分，不能小于 0。
- `stock` 必填，整数，不能小于 0。
- `tags` 是字符串数组。
- `created_at` 和 `updated_at` 是日期。

写出：

```javascript
db.createCollection("products_validated", {
  validator: {
    $jsonSchema: {
      // ...
    }
  }
})
```

然后测试一条正确插入和一条错误插入。

## 12. 本节小结

你需要记住：

- MongoDB 灵活 Schema 不等于没有结构约束。
- Schema Validation 可以拒绝明显错误的数据。
- Go 校验和数据库校验应该配合使用。
- 必填、类型、枚举、范围适合放 Schema Validation。
- 复杂业务规则仍然放应用层。
- 修改已有集合校验规则前要检查历史数据。

