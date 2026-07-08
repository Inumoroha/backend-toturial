# 02. 分片集群组件：mongos、shard、config server

MongoDB 分片集群由多个组件组成。理解它们的职责，是学习分片的第一步。

## 1. 分片集群结构

典型结构：

```text
Application
  |
  |-- mongos
        |
        |-- config server replica set
        |
        |-- shard 1 replica set
        |-- shard 2 replica set
        |-- shard 3 replica set
```

应用通常连接 `mongos`，而不是直接连接 shard。

## 2. shard

shard 是存储数据的分片。

生产环境中，每个 shard 通常是一个复制集：

```text
shard1: rs-shard1
shard2: rs-shard2
shard3: rs-shard3
```

这样每个 shard 自身也有高可用。

## 3. mongos

`mongos` 是查询路由器。

职责：

- 接收应用请求。
- 根据 shard key 和 config metadata 判断请求发往哪些 shard。
- 聚合多个 shard 的结果。
- 对应用隐藏底层分片细节。

`mongos` 本身不保存业务数据。

应用连接串通常指向多个 mongos：

```text
mongodb://mongos1:27017,mongos2:27017
```

## 4. config server

config server 保存分片集群元数据，例如：

- 哪些集合被分片。
- shard key 是什么。
- chunk 分布在哪些 shard。
- 集群配置。

config server 通常也是复制集。

它非常关键，不能当普通业务库使用。

## 5. 数据如何分布

分片集合会按 shard key 划分成 chunk。

例如 shard key 是：

```javascript
{ user_id: 1 }
```

MongoDB 会把不同范围或哈希区间的数据分配到不同 shard。

## 6. 应用是否需要知道数据在哪个 shard

通常不需要。

应用只连接 mongos：

```text
App -> mongos -> shard
```

但应用必须知道：

- 查询最好包含 shard key。
- shard key 选择会影响查询路由。
- 不包含 shard key 的查询可能广播到所有 shard。

## 7. 分片和复制集的关系

复制集解决：

```text
高可用和数据冗余。
```

分片解决：

```text
水平扩展容量和吞吐。
```

生产分片集群通常是：

```text
多个复制集组成的分片集群。
```

不是二选一，而是组合使用。

## 8. 本节练习

画出一个包含：

- 2 个 mongos。
- 3 个 shard。
- 每个 shard 3 节点复制集。
- 3 节点 config server。

的架构图，并说明每个组件的作用。

## 9. 本节小结

你需要记住：

- shard 存储数据。
- mongos 负责查询路由。
- config server 保存元数据。
- 应用通常连接 mongos。
- 每个 shard 通常是复制集。
- 分片解决水平扩展，复制集解决高可用。

## 10. 一次请求在分片集群中的路径

假设 Go 服务执行：

```go
filter := bson.M{
    "tenant_id": tenantID,
    "created_at": bson.M{"$gte": start, "$lt": end},
}

cursor, err := events.Find(ctx, filter)
```

如果 `tenant_id` 是 shard key 的一部分，请求路径大致是：

```text
Go 服务
  -> mongos
  -> 根据 config server 元数据判断目标 chunk
  -> 路由到对应 shard
  -> shard 内部执行查询
  -> 结果返回 mongos
  -> mongos 合并结果
  -> 返回 Go 服务
```

如果查询条件不包含 shard key，mongos 无法判断目标 shard，只能把请求发给多个 shard，再合并结果。这就是后面会讲的 scatter-gather。

## 11. 组件职责边界

### mongos 不存业务数据

mongos 是无状态路由进程。它可以水平扩展，应用通常配置多个 mongos 地址：

```text
mongodb://mongos1:27017,mongos2:27017,mongos3:27017/app
```

Go Driver 会在连接池中管理这些地址。生产里不要只配置一个 mongos，否则 mongos 自己会变成入口单点。

### shard 通常是复制集

生产 shard 不应该是单节点。一个 shard 通常是一个复制集：

```text
shard01: shard01-a, shard01-b, shard01-c
shard02: shard02-a, shard02-b, shard02-c
```

这样每个数据分片内部仍然具备高可用能力。

### config server 也必须高可用

config server 保存元数据，例如：

- 数据库是否开启分片。
- 集合的 shard key。
- chunk 范围。
- chunk 位于哪个 shard。
- zone 配置。

如果 config server 不健康，分片集群的管理操作和路由元数据刷新都会受影响。

## 12. 开发环境与生产环境差异

学习时，你可能会在一台机器上启动多个容器模拟：

```text
1 个 config server 副本集
2 个 shard 副本集
1-2 个 mongos
```

但生产中要注意：

- config server、mongos、shard 不要随意混部。
- mongos 应靠近应用部署，减少网络延迟。
- shard 节点要关注磁盘、IOPS、内存和网络。
- config server 要重点关注稳定性，而不是承载业务查询。
- 运维操作要先在测试集群演练。

## 13. 常见错误理解

### 错误 1：以为 mongos 越多越快

mongos 多可以提升入口可用性和并发承载，但真正慢的是 shard 内查询或 scatter-gather 时，增加 mongos 不会根治问题。

### 错误 2：应用可以直接连接 shard

业务应用应该连接 mongos。直接连接 shard 会绕过分片路由，容易读不到完整数据，也破坏架构边界。

### 错误 3：config server 是普通配置文件

config server 是分片集群核心元数据存储，不是简单配置文件。它必须稳定、备份、监控。

## 14. Go 服务连接建议

连接分片集群时，Repository 层不应该感知某个 shard 地址，只应该依赖统一的 `mongo.Client`：

```go
client, err := mongo.Connect(options.Client().
    ApplyURI("mongodb://mongos1:27017,mongos2:27017,mongos3:27017/app").
    SetServerSelectionTimeout(5 * time.Second).
    SetConnectTimeout(5 * time.Second))
if err != nil {
    return err
}
```

同时建议：

- 把连接字符串放在配置中心或环境变量。
- 配置多个 mongos。
- 给每个请求设置 context timeout。
- 慢查询日志中记录业务查询条件。
- 在代码评审中检查是否带 shard key。

## 15. 运维检查命令

在 mongosh 中可以用：

```javascript
sh.status()
```

查看分片状态、数据库分片状态、集合 shard key、chunk 分布等信息。

也可以查看当前连接是不是 mongos：

```javascript
db.hello()
```

## 16. 官方文档延伸阅读

- [Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [sh.status()](https://www.mongodb.com/docs/manual/reference/method/sh.status/)
