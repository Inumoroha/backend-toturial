# 02. 连接 MongoDB 与 Client 生命周期

本节学习如何用 Go Driver v2 创建 `mongo.Client`、验证连接、选择数据库和集合，以及为什么一个服务进程通常应该复用一个 Client。

## 1. Client、Database、Collection 的关系

在 Go Driver 中：

```text
mongo.Client
  |
  |-- mongo.Database
        |
        |-- mongo.Collection
```

对应 MongoDB：

```text
MongoDB 部署
  |
  |-- database
        |
        |-- collection
```

示例：

```go
client.Database("go_mongodb_stage2").Collection("users")
```

这行代码不会立即访问数据库，它只是拿到一个集合句柄。真正访问数据库的是后续的 `InsertOne`、`FindOne`、`Find` 等操作。

## 2. 最小连接示例

创建 `internal/database/mongo.go`：

```go
package database

import (
    "context"
    "time"

    "go.mongodb.org/mongo-driver/v2/mongo"
    "go.mongodb.org/mongo-driver/v2/mongo/options"
    "go.mongodb.org/mongo-driver/v2/mongo/readpref"
)

func Connect(ctx context.Context, uri string) (*mongo.Client, error) {
    client, err := mongo.Connect(options.Client().ApplyURI(uri))
    if err != nil {
        return nil, err
    }

    pingCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    if err := client.Ping(pingCtx, readpref.Primary()); err != nil {
        _ = client.Disconnect(context.Background())
        return nil, err
    }

    return client, nil
}
```

要点：

- `mongo.Connect` 创建 Client，但不代表 MongoDB 一定可达。
- 官方 API 文档说明，验证部署是否可达应调用 `Client.Ping`。
- 如果 Ping 失败，应释放已经创建的 Client。

## 3. 在 main 中使用

修改 `cmd/api/main.go`：

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "mongodb-go-stage2/internal/config"
    "mongodb-go-stage2/internal/database"
)

func main() {
    cfg := config.Load()

    ctx, cancel := context.WithTimeout(context.Background(), cfg.MongoTimeout)
    defer cancel()

    client, err := database.Connect(ctx, cfg.MongoURI)
    if err != nil {
        log.Fatalf("connect mongodb: %v", err)
    }

    defer func() {
        shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer shutdownCancel()

        if err := client.Disconnect(shutdownCtx); err != nil {
            log.Printf("disconnect mongodb: %v", err)
        }
    }()

    fmt.Println("mongodb connected")
}
```

运行：

```powershell
go run .\cmd\api
```

如果输出：

```text
mongodb connected
```

说明连接成功。

## 4. 为什么不能每个请求都创建 Client

`mongo.Client` 内部维护连接池和监控连接。

官方连接池文档建议：为了效率，每个进程创建一次 Client，并复用于所有操作。每个请求都创建新 Client 会增加延迟和连接数量。

错误思路：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    client, _ := mongo.Connect(options.Client().ApplyURI(uri))
    defer client.Disconnect(context.Background())
    // 每个请求都创建和关闭 Client，不推荐
}
```

推荐思路：

```text
服务启动
  -> 创建一个 mongo.Client
  -> 注入 Repository
  -> 所有请求复用这个 Client
  -> 服务退出时 Disconnect
```

## 5. Database 和 Collection 句柄

可以在 Repository 中保存集合句柄：

```go
type UserRepository struct {
    collection *mongo.Collection
}

func NewUserRepository(db *mongo.Database) *UserRepository {
    return &UserRepository{
        collection: db.Collection("users"),
    }
}
```

在 main 中：

```go
db := client.Database(cfg.MongoDatabase)
userRepo := user.NewUserRepository(db)
```

这样业务模块不需要知道 MongoDB URI，也不需要自己创建 Client。

## 6. Ping 放在哪里

建议：

- 服务启动时 Ping 一次，确认启动依赖可用。
- 健康检查接口可以做轻量 Ping，但不要过于频繁。
- 不要每次业务操作前都 Ping，这会增加额外开销。

启动时 Ping 的目的：

- URI 错误时尽早失败。
- 账号密码错误时尽早失败。
- MongoDB 未启动时尽早失败。

## 7. Disconnect 放在哪里

服务退出时调用：

```go
client.Disconnect(ctx)
```

一般放在 `main` 的 defer 或优雅关闭逻辑里。

不要在每次 Repository 方法里 Disconnect：

```go
func (r *UserRepository) Create(ctx context.Context, user User) error {
    defer r.client.Disconnect(ctx) // 错误
}
```

Repository 只负责执行数据操作，不负责关闭全局 Client。

## 8. 连接失败排查

### MongoDB 容器未启动

检查：

```powershell
docker ps
docker start mongodb-learning
```

### URI 错误

确认：

```text
mongodb://root:example@localhost:27017/?authSource=admin
```

重点检查：

- 用户名。
- 密码。
- 端口。
- `authSource=admin`。

### 端口不是 27017

如果你第 0 阶段改成了 `27018`，URI 也要改：

```text
mongodb://root:example@localhost:27018/?authSource=admin
```

### Ping 超时

可能原因：

- MongoDB 没启动。
- URI 主机错误。
- 防火墙或网络阻断。
- context timeout 太短。

## 9. 本节验收

你应该能做到：

- 创建 `database.Connect` 函数。
- 在 `main` 中连接 MongoDB。
- 调用 `Ping` 验证连接。
- 在程序退出时 `Disconnect`。
- 解释为什么 `mongo.Client` 要复用。
- 解释 `Client`、`Database`、`Collection` 的关系。

