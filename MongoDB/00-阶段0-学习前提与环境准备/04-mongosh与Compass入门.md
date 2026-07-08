# 04. mongosh 与 Compass 入门

本节学习两个工具：

- `mongosh`：命令行工具，适合练习、脚本化、排障。
- MongoDB Compass：图形化工具，适合观察数据结构、索引和查询结果。

两个都要会。只会图形化工具不够，只会命令行也会降低观察效率。

## 1. 用 mongosh 连接本地 MongoDB

确保容器正在运行：

```powershell
docker ps
```

连接：

```powershell
mongosh "mongodb://root:example@localhost:27017/?authSource=admin"
```

连接成功后，提示符会变成类似：

```text
test>
```

这表示你当前位于 `test` 数据库上下文。

## 2. mongosh 基本命令

查看数据库：

```javascript
show dbs
```

查看当前数据库：

```javascript
db
```

切换数据库：

```javascript
use mongodb_learning
```

查看集合：

```javascript
show collections
```

清屏：

```javascript
cls
```

退出：

```javascript
exit
```

## 3. 创建第一批测试数据

切换数据库：

```javascript
use mongodb_learning
```

插入用户：

```javascript
db.users.insertMany([
  {
    name: "Alice",
    email: "alice@example.com",
    age: 25,
    roles: ["user"],
    profile: {
      city: "Shanghai",
      company: "Example Inc"
    },
    created_at: new Date()
  },
  {
    name: "Bob",
    email: "bob@example.com",
    age: 30,
    roles: ["user", "admin"],
    profile: {
      city: "Beijing",
      company: "Demo Tech"
    },
    created_at: new Date()
  },
  {
    name: "Charlie",
    email: "charlie@example.com",
    age: 19,
    roles: ["user"],
    profile: {
      city: "Hangzhou",
      company: "Student"
    },
    created_at: new Date()
  }
])
```

查询所有用户：

```javascript
db.users.find()
```

格式化输出：

```javascript
db.users.find().pretty()
```

## 4. 常用查询入门

按字段查询：

```javascript
db.users.find({ name: "Alice" })
```

查询年龄大于 20：

```javascript
db.users.find({ age: { $gt: 20 } })
```

查询城市：

```javascript
db.users.find({ "profile.city": "Shanghai" })
```

查询数组包含某个元素：

```javascript
db.users.find({ roles: "admin" })
```

只返回部分字段：

```javascript
db.users.find(
  { age: { $gt: 20 } },
  { name: 1, email: 1, age: 1, _id: 0 }
)
```

排序：

```javascript
db.users.find().sort({ age: -1 })
```

限制返回数量：

```javascript
db.users.find().limit(2)
```

## 5. 更新和删除入门

更新一个用户：

```javascript
db.users.updateOne(
  { email: "alice@example.com" },
  { $set: { age: 26, updated_at: new Date() } }
)
```

给数组追加角色：

```javascript
db.users.updateOne(
  { email: "alice@example.com" },
  { $addToSet: { roles: "editor" } }
)
```

删除一个用户：

```javascript
db.users.deleteOne({ email: "charlie@example.com" })
```

删除前要先查：

```javascript
db.users.find({ email: "charlie@example.com" })
```

真实项目中，大批量删除必须非常谨慎。学习阶段也要养成先查再删的习惯。

## 6. 用 Compass 连接 MongoDB

打开 MongoDB Compass。

连接 URI 填：

```text
mongodb://root:example@localhost:27017/?authSource=admin
```

点击连接后，你应该能看到数据库列表。

找到：

```text
mongodb_learning
```

再找到：

```text
users
```

你应该能看到刚才通过 `mongosh` 插入的用户文档。

## 7. Compass 中做基本查询

进入 `mongodb_learning.users` 集合后，在 Filter 输入：

```json
{ "age": { "$gt": 20 } }
```

点击 Find。

Projection 可以输入：

```json
{ "name": 1, "email": 1, "age": 1, "_id": 0 }
```

Sort 可以输入：

```json
{ "age": -1 }
```

Limit 可以填：

```text
2
```

这和 `mongosh` 中的查询是一回事，只是 Compass 把输入区域图形化了。

## 8. Compass 中查看文档结构

Compass 的 Schema 功能可以帮助你观察集合字段。

你可以看到：

- 哪些字段出现频率高。
- 字段类型是什么。
- 某些字段是否经常缺失。
- 数组字段和嵌套字段的大致结构。

但注意：MongoDB 是灵活 Schema，不代表可以随便乱放字段。

Go 后端项目中通常仍然需要通过结构体、校验逻辑、Schema Validation 保持数据结构稳定。

## 9. 命令行和图形化工具怎么配合

推荐习惯：

- 用 `mongosh` 练语法。
- 用 Compass 看数据。
- 用 `mongosh` 做批量实验。
- 用 Compass 看索引、文档结构和 explain。
- 遇到线上问题时，先用命令行复现关键查询。

## 10. 本节练习

在 `mongodb_learning` 数据库中完成：

1. 插入 5 个用户。
2. 至少 2 个用户有 `admin` 角色。
3. 至少 2 个用户来自同一个城市。
4. 查询年龄大于 20 的用户。
5. 查询某个城市的用户。
6. 查询有 `admin` 角色的用户。
7. 给一个用户增加 `editor` 角色。
8. 删除一个测试用户。
9. 在 Compass 中复现第 4、5、6 条查询。

## 11. 本节验收

你应该能回答：

- `show dbs` 和 `show collections` 分别做什么？
- `use mongodb_learning` 是否会立刻创建数据库？
- `find` 第二个参数是什么？
- 如何查询嵌套字段？
- 如何查询数组中包含某个值？
- Compass 的 Filter 和 mongosh 的查询语法有什么关系？

提示：`use` 一个数据库并不会立刻创建它，只有写入数据后才会真正出现。

