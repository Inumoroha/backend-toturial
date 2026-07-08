# 第 2 阶段：Go Driver v2 与后端集成

本阶段把第 1 阶段学过的 MongoDB 基础操作搬进 Go 后端项目。你要学会使用官方 Go Driver v2 连接 MongoDB、定义 BSON 模型、封装 Repository、处理 Cursor、处理错误、控制超时，并为数据访问层写集成测试。

这一阶段非常关键。很多人会写 `mongosh`，但一到 Go 项目里就出现连接乱建、context 不传、错误不分层、BSON tag 写错、Cursor 不关闭、测试不可重复等问题。本阶段就是把这些坑提前填掉。

## 本阶段目标

完成本阶段后，你应该能做到：

- 使用 `go.mongodb.org/mongo-driver/v2` 安装官方 Go Driver v2。
- 理解 `mongo.Client`、`mongo.Database`、`mongo.Collection` 的关系。
- 正确创建和复用 `mongo.Client`。
- 使用 `context.Context` 控制连接、查询和写入超时。
- 使用 `bson.D`、`bson.M`、`bson.A` 和结构体 BSON tag。
- 使用 `bson.ObjectID`、`bson.NewObjectID()`、`bson.ObjectIDFromHex()`。
- 使用 `InsertOne`、`FindOne`、`Find`、`UpdateOne`、`DeleteOne`。
- 正确处理 Cursor：遍历、Decode、All、Close、Err。
- 正确处理 `mongo.ErrNoDocuments`、重复键错误、超时错误。
- 封装一个可维护的 `UserRepository`。
- 为 Repository 写可重复执行的集成测试。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-Go项目初始化与驱动安装.md` | 初始化 Go 项目，安装 Go Driver v2 |
| 02 | `02-连接MongoDB与Client生命周期.md` | 学习连接、Ping、Disconnect、Client 复用 |
| 03 | `03-BSON类型与Struct标签.md` | 学习 BSON 类型、ObjectID、struct tag |
| 04 | `04-Repository模式与项目分层.md` | 学习后端分层和 Repository 封装 |
| 05 | `05-插入与单文档查询.md` | 用 Go 实现 InsertOne、FindOne |
| 06 | `06-多文档查询Cursor排序分页.md` | 用 Go 实现 Find、Cursor、排序分页 |
| 07 | `07-更新删除与Upsert.md` | 用 Go 实现 UpdateOne、DeleteOne、Upsert |
| 08 | `08-错误处理与业务错误转换.md` | 处理未找到、重复键、超时、网络错误 |
| 09 | `09-Context超时连接池与优雅关闭.md` | 学习 context、连接池配置和关闭 |
| 10 | `10-集成测试与测试数据管理.md` | 为 MongoDB Repository 写集成测试 |
| 11 | `11-阶段练习-UserRepository.md` | 阶段项目练习和验收 |

## 本阶段统一约定

本阶段使用本地 MongoDB：

```text
mongodb://root:example@localhost:27017/?authSource=admin
```

数据库名：

```text
go_mongodb_stage2
```

项目目录建议：

```text
mongodb-go-stage2/
  cmd/
    api/
      main.go
  internal/
    config/
    database/
    user/
      model.go
      repository.go
      errors.go
      repository_test.go
```

学习阶段可以先写命令行程序，不急着接 HTTP 框架。先把数据访问层写稳，再接 Gin、Echo、Fiber 或标准库 HTTP 都不迟。

## 官方资料

本阶段主要参考：

- Go Driver Getting Started：<https://www.mongodb.com/docs/drivers/go/current/get-started/>
- Work with BSON：<https://www.mongodb.com/docs/drivers/go/current/data-formats/bson/>
- Context Package：<https://www.mongodb.com/docs/drivers/go/current/context/>
- Connection Pools：<https://www.mongodb.com/docs/drivers/go/current/connect/connection-options/connection-pools/>
- Find Documents：<https://www.mongodb.com/docs/drivers/go/current/crud/query/retrieve/>
- Upsert：<https://www.mongodb.com/docs/drivers/go/current/crud/update/upsert/>
- Go Driver API：<https://pkg.go.dev/go.mongodb.org/mongo-driver/v2/mongo>
- BSON API：<https://pkg.go.dev/go.mongodb.org/mongo-driver/v2/bson>

## 一个重要提醒

Go Driver v2 的导入路径带 `/v2`：

```go
import (
    "go.mongodb.org/mongo-driver/v2/bson"
    "go.mongodb.org/mongo-driver/v2/mongo"
    "go.mongodb.org/mongo-driver/v2/mongo/options"
)
```

不要把 v1 和 v2 混用。尤其是 ObjectID，v2 推荐使用：

```go
bson.ObjectID
bson.NewObjectID()
bson.ObjectIDFromHex(...)
```

不要在 v2 项目里混入旧教程中的：

```go
go.mongodb.org/mongo-driver/bson/primitive
```

混用大版本很容易导致 ObjectId 编码异常、类型不匹配或依赖混乱。

