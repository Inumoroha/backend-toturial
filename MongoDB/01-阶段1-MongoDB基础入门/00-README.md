# 第 1 阶段：MongoDB 基础入门

本阶段正式进入 MongoDB。目标不是背命令，而是建立 MongoDB 文档数据库的基本操作能力：能理解文档模型，能用 `mongosh` 完成 CRUD，能查询嵌套字段和数组字段，并能用一个小型博客系统把这些知识串起来。

## 本阶段目标

完成本阶段后，你应该能做到：

- 解释 MongoDB、关系型数据库、文档数据库的差异。
- 说清楚 database、collection、document 的关系。
- 理解 BSON 和 JSON 的区别。
- 理解 `_id` 和 ObjectId 的作用。
- 熟练使用 `insertOne`、`insertMany`。
- 熟练使用 `find`、过滤条件、投影、排序、分页。
- 熟练使用 `updateOne`、`updateMany` 和常见更新操作符。
- 熟练使用 `deleteOne`、`deleteMany`，并知道删除操作的风险。
- 能查询嵌套文档和数组字段。
- 能用 `mongosh` 完成一组博客系统数据练习。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-文档数据库与BSON.md` | 理解文档数据库、JSON、BSON |
| 02 | `02-数据库集合文档与_id.md` | 掌握 database、collection、document、`_id` |
| 03 | `03-插入文档.md` | 学习 `insertOne`、`insertMany` |
| 04 | `04-查询文档.md` | 学习 `find`、过滤、投影、排序、分页 |
| 05 | `05-更新文档.md` | 学习 `$set`、`$inc`、`$push`、`$pull` 等更新 |
| 06 | `06-删除文档.md` | 学习删除操作和安全习惯 |
| 07 | `07-数组与嵌套文档查询.md` | 学习点语法、数组匹配、`$elemMatch` |
| 08 | `08-阶段练习-博客系统数据.md` | 使用博客系统完成完整 CRUD 练习 |
| 09 | `09-阶段验收与命令速查.md` | 自测题、验收标准和常用命令表 |

## 学习前准备

请确认第 0 阶段已经完成：

- Docker 中的 `mongodb-learning` 容器能启动。
- `mongosh` 可以连接本地 MongoDB。
- MongoDB Compass 可以连接本地 MongoDB。

连接命令：

```powershell
mongosh "mongodb://root:example@localhost:27017/?authSource=admin"
```

本阶段统一使用数据库：

```javascript
use mongodb_stage1
```

如果你之前做过实验，想清空这个数据库：

```javascript
use mongodb_stage1
db.dropDatabase()
```

注意：`dropDatabase()` 会删除当前数据库里的所有数据。只在学习环境中使用。

## 学习方法

每学一个命令，都要问自己四个问题：

1. 这个命令操作的是哪个集合？
2. 过滤条件是什么？
3. 会影响几条文档？
4. 如果这是线上数据，会不会误伤？

尤其是 `updateMany` 和 `deleteMany`，一定要先查询再操作。

推荐操作习惯：

```javascript
// 1. 先查确认范围
db.users.find({ status: "inactive" })

// 2. 再更新或删除
db.users.updateMany(
  { status: "inactive" },
  { $set: { archived: true } }
)
```

## 官方资料

本阶段主要参考：

- MongoDB Documents：<https://www.mongodb.com/docs/manual/core/document/>
- MongoDB CRUD：<https://www.mongodb.com/docs/manual/crud/>
- Query Operators：<https://www.mongodb.com/docs/manual/reference/operator/query/>
- Update Operators：<https://www.mongodb.com/docs/manual/reference/operator/update/>
- mongosh：<https://www.mongodb.com/docs/mongodb-shell/>

