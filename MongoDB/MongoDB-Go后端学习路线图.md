# MongoDB 系统学习路线图：面向 Go 后端工程师

> 目标：不是只会写 CRUD，而是能在 Go 后端项目中正确建模、设计索引、写出可靠查询、处理事务与并发、理解复制集/分片、完成性能优化和线上排障。
>
> 建议周期：12 周。每天 1.5-2 小时，周末做项目和复盘。若时间充裕，可拉长到 16 周；若已有数据库基础，可压缩到 8 周。

## 0. 学习前提

### 你需要具备

- Go 基础：结构体、接口、错误处理、`context.Context`、goroutine、module、单元测试。
- Web 后端基础：HTTP、REST API、认证、分页、日志、配置、Docker。
- 数据库基础：索引、事务、读写一致性、SQL 的基本概念。

### 推荐环境

- Go 版本：使用当前稳定版本即可。
- MongoDB：本地 Docker + MongoDB Atlas 免费集群都建议使用。
- Go Driver：优先学习官方 v2 驱动：`go.mongodb.org/mongo-driver/v2`。
- GUI：MongoDB Compass。
- 工具：Docker、mongosh、Postman/Bruno、Grafana 或 Prometheus 了解即可。

### 最小本地环境

```bash
docker run -d --name mongodb-learning \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=example \
  mongo:latest
```

连接 URI 示例：

```text
mongodb://root:example@localhost:27017/?authSource=admin
```

## 1. 总体路线

```text
第 1 阶段：入门与文档模型
  MongoDB 基础、BSON、Collection、CRUD、mongosh

第 2 阶段：Go 集成
  Go Driver v2、连接池、context、Repository、错误处理、测试

第 3 阶段：建模与查询能力
  文档建模、嵌入/引用、聚合管道、分页、排序、数据校验

第 4 阶段：索引与性能
  单字段/复合/唯一/TTL/文本索引、explain、慢查询、容量估算

第 5 阶段：可靠性与分布式
  复制集、读写关注、事务、变更流、备份恢复、分片

第 6 阶段：工程化实战
  项目落地、压测、监控、故障演练、面试复盘
```

## 2. 12 周学习计划

| 周次 | 主题 | 核心目标 | 产出 |
| --- | --- | --- | --- |
| 第 1 周 | MongoDB 基础 | 理解文档数据库、BSON、库/集合/文档、基本 CRUD | 用 mongosh 完成一组博客数据 CRUD |
| 第 2 周 | 查询语法 | 掌握过滤、排序、投影、分页、更新操作符、数组查询 | 写 30 个查询练习 |
| 第 3 周 | Go Driver v2 | 用 Go 正确连接 MongoDB，完成 CRUD 封装 | Go 版用户管理 Repository |
| 第 4 周 | Go 工程实践 | context、连接池、超时、错误处理、单元测试 | 带测试的 MongoDB 数据访问层 |
| 第 5 周 | 文档建模 | 学会嵌入、引用、反范式、关系迁移思维 | 设计电商/内容系统数据模型 |
| 第 6 周 | 聚合管道 | 掌握 `$match`、`$group`、`$lookup`、`$unwind`、`$project` | 订单统计 API |
| 第 7 周 | 索引基础 | 单字段、复合、唯一、稀疏/部分、TTL 索引 | 为项目建立索引清单 |
| 第 8 周 | 性能优化 | explain、慢查询、分页优化、批量写入、容量估算 | 优化 5 条慢查询并写报告 |
| 第 9 周 | 事务与一致性 | 单文档原子性、多文档事务、读写关注 | 钱包转账/库存扣减 Demo |
| 第 10 周 | 复制集与高可用 | 主从复制、选举、读偏好、故障切换 | 本地搭建三节点复制集 |
| 第 11 周 | 分片与大规模数据 | shard key、chunk、balancer、热点问题 | 设计一个千万级日志系统方案 |
| 第 12 周 | 综合项目 | 完成可展示项目、压测、监控、复盘 | 一个可写进简历的 Go + MongoDB 项目 |

## 3. 分阶段详细路线

### 第 1 阶段：MongoDB 基础入门

学习重点：

- MongoDB 和关系型数据库的差异。
- Document、Collection、Database 的概念。
- BSON 与 JSON 的区别。
- `_id`、`ObjectID`、时间戳、嵌套文档、数组字段。
- `insertOne`、`insertMany`、`find`、`updateOne`、`updateMany`、`deleteOne`、`deleteMany`。
- 常用查询条件：`$eq`、`$ne`、`$gt`、`$gte`、`$lt`、`$lte`、`$in`、`$nin`、`$exists`、`$regex`。
- 更新操作符：`$set`、`$unset`、`$inc`、`$push`、`$pull`、`$addToSet`。

练习任务：

- 建一个 `blog` 数据库。
- 设计 `users`、`posts`、`comments` 三个集合。
- 完成用户注册、发布文章、评论文章、查询文章列表、修改文章状态。
- 用 `mongosh` 写出不少于 30 条查询。

验收标准：

- 能解释为什么 MongoDB 的一条记录叫 document。
- 能熟练写基本 CRUD。
- 能区分数组字段查询和嵌套文档查询。
- 能解释 `_id` 的作用，以及为什么不要随便修改它。

### 第 2 阶段：Go Driver v2 与后端集成

学习重点：

- 安装官方 Go Driver v2。
- `mongo.Connect`、`Client`、`Database`、`Collection` 的关系。
- `context.Context` 超时与取消。
- 连接池配置、Ping、优雅关闭。
- `bson.D`、`bson.M`、`bson.A`、struct tag。
- `FindOne`、`Find`、`InsertOne`、`UpdateOne`、`DeleteOne`。
- Cursor 处理、Decode、`cursor.All`。
- 常见错误：`mongo.ErrNoDocuments`、重复键错误、超时错误。

建议目录结构：

```text
internal/
  config/
  database/
  user/
    model.go
    repository.go
    service.go
    handler.go
```

Go 模型示例：

```go
type User struct {
    ID        bson.ObjectID `bson:"_id,omitempty" json:"id"`
    Email     string        `bson:"email" json:"email"`
    Name      string        `bson:"name" json:"name"`
    CreatedAt time.Time     `bson:"created_at" json:"created_at"`
    UpdatedAt time.Time     `bson:"updated_at" json:"updated_at"`
}
```

练习任务：

- 用 Go 写一个 `UserRepository`。
- 实现 `CreateUser`、`FindByID`、`FindByEmail`、`UpdateProfile`、`DeleteUser`。
- 所有数据库操作都必须传入 `context.Context`。
- 给每个公开方法写单元测试或集成测试。

验收标准：

- 能解释为什么不能到处创建新的 `mongo.Client`。
- 能正确处理 `context timeout`。
- 能说明 `bson.D` 和 `bson.M` 的使用差异。
- 能把 MongoDB 错误转换成业务错误。

### 第 3 阶段：文档建模

学习重点：

- 嵌入式模型 vs 引用式模型。
- 一对一、一对多、多对多在 MongoDB 中如何设计。
- 读多写少与写多读少的建模差异。
- 反范式设计：为了查询效率适当冗余。
- 文档大小限制、数组无限增长问题。
- 数据生命周期：软删除、归档、冷热数据。
- Schema Validation。

建模原则：

- 优先围绕查询场景设计，而不是照搬关系型数据库表结构。
- 经常一起读取的数据可以嵌入。
- 会无限增长的数组不要直接塞进一个文档。
- 需要独立查询、独立更新、独立生命周期的数据适合拆集合。
- 冗余字段要有同步策略。

练习任务：

- 设计一个电商订单系统：
  - 用户
  - 商品
  - 购物车
  - 订单
  - 订单明细
  - 支付记录
  - 库存流水
- 分别给出“关系型思路”和“MongoDB 思路”的模型。
- 写出每个集合的 5 个核心查询场景。

验收标准：

- 能解释什么时候嵌入，什么时候引用。
- 能识别“无限增长数组”这种常见坑。
- 能围绕业务查询反推集合设计。

### 第 4 阶段：聚合管道

学习重点：

- 聚合管道基本思想。
- 常用 stage：`$match`、`$project`、`$group`、`$sort`、`$limit`、`$skip`。
- 数组处理：`$unwind`。
- 集合关联：`$lookup`。
- 字段计算：`$addFields`、`$set`。
- 时间统计：按日、按月、按用户、按状态聚合。
- Go 中构造 aggregation pipeline。

练习任务：

- 订单金额按天统计。
- 用户消费排行榜。
- 商品销量排行榜。
- 文章阅读量、点赞数、评论数聚合。
- 查询用户最近 30 天活跃情况。

验收标准：

- 能独立写出常见报表类聚合。
- 知道 `$match` 尽量前置以减少后续数据量。
- 知道 `$lookup` 不是万能替代 join，需要考虑数据量和索引。

### 第 5 阶段：索引与性能优化

学习重点：

- 为什么没有索引会 collection scan。
- 单字段索引、复合索引、唯一索引、TTL 索引、部分索引、文本索引。
- 复合索引字段顺序。
- ESR 原则：Equality、Sort、Range。
- `explain("executionStats")`。
- 慢查询分析。
- 覆盖索引。
- 分页优化：避免大偏移量 `skip`。
- 批量写入：`BulkWrite`。
- 索引不是越多越好，写入也有成本。

练习任务：

- 给用户邮箱建立唯一索引。
- 给订单列表建立复合索引：用户、状态、创建时间。
- 给验证码/临时 token 建立 TTL 索引。
- 用 explain 对比有索引和无索引的查询。
- 把基于 `skip + limit` 的深分页改为基于游标的分页。

验收标准：

- 能看懂 `COLLSCAN`、`IXSCAN`、`nReturned`、`totalDocsExamined`。
- 能根据查询条件设计复合索引。
- 能解释索引为什么会拖慢写入。
- 能说出深分页的性能问题和替代方案。

### 第 6 阶段：事务、一致性与并发

学习重点：

- MongoDB 单文档原子性。
- 多文档事务适用场景。
- 事务的成本和限制。
- session、callback API、commit、abort。
- readConcern、writeConcern、readPreference。
- 幂等设计。
- 乐观锁：版本号或更新时间。
- 唯一索引防并发重复。

练习任务：

- 实现钱包转账：
  - A 扣款
  - B 加款
  - 写交易流水
  - 全部成功才提交
- 实现库存扣减：
  - 防止库存扣成负数
  - 防止重复扣减
  - 写库存流水

验收标准：

- 能解释“单文档原子性”和“多文档事务”的区别。
- 能判断什么时候不该用事务。
- 能用唯一索引和幂等键解决重复请求。

### 第 7 阶段：复制集、高可用与备份恢复

学习重点：

- 复制集架构：primary、secondary、arbiter。
- oplog。
- 选举机制和故障切换。
- readPreference：primary、secondary、nearest 等。
- writeConcern：`w:1`、`majority`。
- 备份与恢复：mongodump、mongorestore、Atlas Backup。
- Change Streams 基础。

练习任务：

- 用 Docker Compose 搭建三节点复制集。
- 停掉 primary，观察重新选主。
- 测试 `writeConcern: majority`。
- 使用 Change Streams 监听订单状态变更。
- 做一次备份和恢复演练。

验收标准：

- 能解释 MongoDB 如何实现高可用。
- 能说明 secondary 读的收益和风险。
- 能解释为什么重要写入建议使用 majority。

### 第 8 阶段：分片与大规模数据

学习重点：

- 分片解决什么问题。
- shard、mongos、config server。
- shard key 的选择。
- chunk、balancer、数据迁移。
- 范围分片和哈希分片。
- 热点写入问题。
- 唯一索引和分片键的关系。
- 分片集合的查询路由。

练习任务：

- 设计一个日志/埋点系统：
  - 每天千万级写入
  - 按用户查询
  - 按时间范围查询
  - 支持数据过期
- 给出 shard key 方案，并说明优缺点。
- 设计索引和 TTL 策略。

验收标准：

- 能解释 shard key 为什么非常关键。
- 能识别单调递增字段作为 shard key 的热点风险。
- 能区分需要分片和只需要优化索引/硬件的场景。

## 4. Go 后端工程师必须掌握的 MongoDB 工程能力

### 连接管理

- 一个服务通常复用一个 `mongo.Client`。
- 启动时初始化连接，关闭服务时优雅断开。
- 所有数据库操作都带 `context`。
- 为请求设置合理超时，不让数据库调用无限阻塞。

### 数据访问层设计

- Handler 不直接拼 MongoDB 查询。
- Repository 负责数据库细节。
- Service 负责业务逻辑、事务和幂等。
- Model 区分数据库模型和 API DTO。

### 错误处理

- `ErrNoDocuments` 转成业务层的“资源不存在”。
- duplicate key 转成“邮箱已存在/订单已存在”。
- timeout 转成可观测的系统错误。
- 不把底层数据库错误原样暴露给前端。

### 测试策略

- Repository 层建议使用集成测试。
- 使用 Docker 启动真实 MongoDB 做测试。
- 为唯一索引、事务、并发扣减写专项测试。
- 测试数据每次运行前清理，避免互相污染。

### 线上规范

- 所有核心查询都必须有索引评审。
- 上线前用真实规模数据跑 explain。
- 慢查询要能定位到接口、查询条件和调用链。
- 大批量更新/删除要分批执行。
- 重要集合要有备份和恢复演练。

## 5. 推荐项目

### 项目 1：博客/内容系统

适合阶段：第 1-4 周。

核心功能：

- 用户注册登录。
- 发布文章。
- 评论、点赞、收藏。
- 文章列表分页。
- 按标签、作者、时间查询。

重点训练：

- 基础 CRUD。
- 嵌套文档和数组字段。
- Go Repository。
- 基础索引。

### 项目 2：电商订单系统

适合阶段：第 5-9 周。

核心功能：

- 商品管理。
- 购物车。
- 下单。
- 库存扣减。
- 支付流水。
- 订单状态流转。

重点训练：

- 文档建模。
- 复合索引。
- 聚合统计。
- 事务。
- 幂等。

### 项目 3：高并发日志/埋点系统

适合阶段：第 10-12 周。

核心功能：

- 批量写入日志。
- 按用户、事件、时间查询。
- TTL 自动过期。
- 聚合统计。
- 压测和慢查询分析。

重点训练：

- BulkWrite。
- TTL 索引。
- 高写入场景建模。
- 分片设计。
- 性能优化。

## 6. 每周学习节奏

建议按下面节奏推进：

| 时间 | 内容 |
| --- | --- |
| 周一 | 学概念，看官方文档 |
| 周二 | 写基础 Demo |
| 周三 | 做查询/建模/索引练习 |
| 周四 | 接入 Go 项目 |
| 周五 | 写测试和整理笔记 |
| 周六 | 做小项目功能 |
| 周日 | 复盘、优化、面试题总结 |

每周复盘问题：

- 这个主题解决什么真实后端问题？
- 在 Go 项目中应该放在哪一层？
- 哪些地方会影响性能？
- 哪些地方容易造成线上事故？
- 我能否用自己的话讲给别人听？

## 7. 面试与实战重点清单

### 初级必须会

- MongoDB 基本概念。
- CRUD。
- Go Driver 连接和查询。
- BSON struct tag。
- 基础索引。
- 简单聚合。

### 中级必须会

- 文档建模取舍。
- 复合索引设计。
- explain 分析。
- 事务和幂等。
- 连接池和超时控制。
- 慢查询排查。
- 备份恢复。

### 高级加分项

- 复制集原理。
- 分片键设计。
- Change Streams。
- 大数据量归档。
- 线上容量评估。
- MongoDB 与 Redis/MySQL/Elasticsearch 的组合使用。
- 根据业务读写模式做数据模型演进。

## 8. 常见坑

- 把 MongoDB 当成“没有表的 MySQL”使用。
- 所有关系都用 `$lookup`，导致查询越来越慢。
- 把无限增长的数组塞进单个文档。
- 没有唯一索引，只靠代码判断重复。
- 不设置 context timeout。
- 每个请求都新建一个 MongoDB client。
- 查询上线前不跑 explain。
- 索引只按单个字段建，没有结合真实查询。
- 深分页一直使用大 `skip`。
- 事务滥用，把本可以单文档原子完成的操作复杂化。
- 没有备份恢复演练。

## 9. 推荐官方资料

以下资料建议作为主线阅读，不要只看二手教程：

- MongoDB Go Driver 文档：<https://www.mongodb.com/docs/drivers/go/current/>
- Go Driver Getting Started：<https://www.mongodb.com/docs/drivers/go/current/get-started/>
- Go Driver Context 指南：<https://www.mongodb.com/docs/drivers/go/current/context/>
- Go Driver BSON 指南：<https://www.mongodb.com/docs/drivers/go/current/data-formats/bson/>
- MongoDB CRUD：<https://www.mongodb.com/docs/manual/crud/>
- MongoDB Indexes：<https://www.mongodb.com/docs/manual/indexes/>
- MongoDB Aggregation：<https://www.mongodb.com/docs/manual/aggregation/>
- MongoDB Transactions：<https://www.mongodb.com/docs/manual/core/transactions/>
- MongoDB Sharding：<https://www.mongodb.com/docs/manual/sharding/>
- MongoDB University：<https://learn.mongodb.com/>

## 10. 最终验收：你应该能做到什么

学完后，你应该能独立完成：

- 用 Go + MongoDB 开发一个完整后端服务。
- 根据业务查询设计集合结构。
- 为核心接口设计合理索引。
- 用 explain 判断查询是否健康。
- 处理唯一约束、并发写入、幂等请求。
- 在需要时使用事务，而不是滥用事务。
- 理解复制集、读写关注、故障切换。
- 对大数据量场景提出分片和归档方案。
- 写出能被团队维护的 MongoDB 数据访问层。

## 11. 建议的简历项目描述

可以按这个方向包装你的项目经历：

```text
基于 Go + MongoDB 设计并实现电商订单服务，包含用户、商品、购物车、订单、库存流水等模块。
根据订单列表、用户订单、商品销量统计等核心查询设计复合索引，并使用 explain 优化慢查询。
通过 MongoDB 事务保证下单、扣减库存、写入流水的一致性，使用唯一索引和幂等键防止重复提交。
使用 BulkWrite 优化批量日志写入，并通过 TTL 索引实现临时数据自动清理。
```

## 12. 学习顺序一句话版

先用 mongosh 熟悉文档和查询，再用 Go Driver 做 CRUD；接着围绕业务查询学习建模和聚合；然后重点攻克索引、explain 和性能；最后补齐事务、复制集、分片、备份恢复和工程化项目。

