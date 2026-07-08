# 1. go-redis/v9 安装与 redis.Client 初始化

第五阶段开始把 Redis 接入 Go 服务。

前面几阶段我们主要在 `redis-cli` 里操作 Redis，理解命令、数据结构、缓存模式。从这一阶段开始，重点变成：如何在 Go 后端项目里稳定、清晰地使用 Redis。

本节先完成第一步：安装 `github.com/redis/go-redis/v9`，初始化 `redis.Client`，并用 `PING` 验证连接。

学完这一节后，你应该能够：

- 在 Go 项目中安装 `go-redis/v9`。
- 使用 `redis.NewClient` 创建 Redis 客户端。
- 理解 `redis.Options` 中最常用的配置。
- 使用 `Ping` 验证 Redis 连接。
- 知道 `redis.Client` 应该复用，而不是每次请求都创建。
- 写出一个简单的 Redis 初始化模块。

---

## 一、准备 Redis 环境

先确保本地 Redis 已经启动。

如果你使用前面阶段的 Docker Compose，可以进入第一阶段目录执行：

```bash
docker compose up -d
```

也可以直接启动一个 Redis 容器：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine
```

验证 Redis：

```bash
docker exec -it redis-dev redis-cli PING
```

返回：

```text
PONG
```

说明 Redis 正常。

---

## 二、创建 Go 项目

创建一个练习目录：

```bash
mkdir redis-go-demo
cd redis-go-demo
go mod init redis-go-demo
```

安装 `go-redis/v9`：

```bash
go get github.com/redis/go-redis/v9
```

查看 `go.mod`，应该能看到类似依赖：

```go
require github.com/redis/go-redis/v9 v9.x.x
```

版本号可能不同，这是正常的。

---

## 三、最小连接示例

创建 `main.go`：

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()

    rdb := redis.NewClient(&redis.Options{
        Addr:     "127.0.0.1:6379",
        Password: "",
        DB:       0,
    })
    defer rdb.Close()

    pong, err := rdb.Ping(ctx).Result()
    if err != nil {
        log.Fatalf("ping redis failed: %v", err)
    }

    fmt.Println(pong)
}
```

运行：

```bash
go run .
```

如果输出：

```text
PONG
```

说明 Go 已经成功连接 Redis。

---

## 四、redis.NewClient 做了什么

核心代码是：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:     "127.0.0.1:6379",
    Password: "",
    DB:       0,
})
```

`redis.NewClient` 会创建一个 Redis 客户端对象。

这个对象不是单次连接，而是带连接池管理能力的客户端。

你应该把它作为长期复用对象，而不是每个请求里创建一个。

不推荐：

```go
func Handler(w http.ResponseWriter, r *http.Request) {
    rdb := redis.NewClient(...)
    defer rdb.Close()
    // 每个请求都创建客户端，错误
}
```

推荐：

```go
func main() {
    rdb := redis.NewClient(...)
    defer rdb.Close()

    // 把 rdb 注入到 handler / service / repository
}
```

---

## 五、Options 常用字段

最基础的配置：

```go
redis.Options{
    Addr:     "127.0.0.1:6379",
    Password: "",
    DB:       0,
}
```

字段说明：

| 字段 | 含义 |
| --- | --- |
| `Addr` | Redis 地址，格式是 `host:port` |
| `Password` | Redis 密码，没有密码就留空 |
| `DB` | Redis 逻辑数据库编号，默认常用 0 |

如果你启动 Redis 时设置了密码：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine redis-server --requirepass 123456
```

Go 里要写：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:     "127.0.0.1:6379",
    Password: "123456",
    DB:       0,
})
```

---

## 六、Ping 是连接健康检查，不是业务命令

`Ping` 常用于启动时验证 Redis 是否可用：

```go
pong, err := rdb.Ping(ctx).Result()
if err != nil {
    return err
}
fmt.Println(pong)
```

但不要在每个业务请求前都 `Ping` 一次。

错误做法：

```go
func GetArticle(ctx context.Context, id int64) {
    rdb.Ping(ctx)
    rdb.Get(ctx, key)
}
```

这样会额外增加一次 Redis 请求，降低性能。

更合理的是：

- 服务启动时 `Ping` 检查。
- 业务请求直接执行需要的 Redis 命令。
- 通过错误处理、日志、监控发现 Redis 异常。

---

## 七、封装初始化函数

真实项目里，不建议把 Redis 初始化散落在 `main.go` 里。

可以封装一个函数：

```go
package cache

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type RedisConfig struct {
    Addr     string
    Password string
    DB       int
}

func NewRedisClient(ctx context.Context, cfg RedisConfig) (*redis.Client, error) {
    rdb := redis.NewClient(&redis.Options{
        Addr:     cfg.Addr,
        Password: cfg.Password,
        DB:       cfg.DB,
    })

    pingCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    if err := rdb.Ping(pingCtx).Err(); err != nil {
        _ = rdb.Close()
        return nil, fmt.Errorf("ping redis: %w", err)
    }

    return rdb, nil
}
```

使用：

```go
rdb, err := cache.NewRedisClient(context.Background(), cache.RedisConfig{
    Addr: "127.0.0.1:6379",
    DB:   0,
})
if err != nil {
    log.Fatal(err)
}
defer rdb.Close()
```

这里用了 `context.WithTimeout`，下一节会专门讲为什么 Redis 操作必须重视 `context`。

---

## 八、redis.Client 的生命周期

`redis.Client` 应该：

- 程序启动时创建。
- 在服务运行期间复用。
- 通过依赖注入传给需要 Redis 的模块。
- 程序退出时关闭。

不要：

- 每次请求创建一个 client。
- 每次 Redis 命令创建一个 client。
- 创建后忘记关闭。
- 把 Redis client 当成全局变量到处乱用又无法测试。

比较清晰的结构是：

```text
main.go
  初始化 redis.Client
  初始化 ArticleCache
  初始化 Handler
```

这样依赖关系明确，也方便写测试。

---

## 九、项目结构建议

一个小项目可以这样组织：

```text
redis-go-demo/
  go.mod
  main.go
  internal/
    cache/
      redis.go
    article/
      model.go
      repository.go
      cache.go
      service.go
```

职责划分：

| 文件 | 职责 |
| --- | --- |
| `internal/cache/redis.go` | 初始化 Redis client |
| `internal/article/cache.go` | 文章缓存读写封装 |
| `internal/article/repository.go` | 数据库访问 |
| `internal/article/service.go` | 业务流程：先缓存，后数据库 |
| `main.go` | 组装依赖，启动 HTTP 服务 |

初学阶段不必一次搭很复杂，但要有意识：Redis 操作应该封装在清晰的模块里。

---

## 十、常见错误

### 1. 每个请求都 NewClient

这会反复创建连接池，浪费资源。

### 2. 忘记 Close

程序退出时应该关闭 client，释放连接资源。

### 3. 每个请求都 Ping

`Ping` 是健康检查，不是每次业务请求的前置步骤。

### 4. 地址写错

Docker 映射如果是 `-p 6380:6379`，Go 里应该连接 `127.0.0.1:6380`。

### 5. 密码漏填

Redis 设置了密码，Go 客户端必须配置 `Password`。

---

## 十一、本节练习

请完成下面练习：

1. 创建一个 Go module。
2. 安装 `github.com/redis/go-redis/v9`。
3. 写一个最小 `main.go`，连接本地 Redis。
4. 使用 `Ping` 输出 `PONG`。
5. 把 Redis 初始化封装成 `NewRedisClient` 函数。
6. 给 `Ping` 增加 2 秒超时。
7. 尝试填错端口，观察错误。
8. 如果 Redis 设置了密码，尝试不填密码，观察错误。
9. 思考为什么不能每个请求都创建 `redis.Client`。
10. 设计你自己的项目目录结构，把 Redis 初始化放到合适位置。

---

## 十二、本节小结

这一节你完成了 Go 接入 Redis 的第一步。

你需要记住：

- Go 推荐使用 `github.com/redis/go-redis/v9` 操作 Redis。
- `redis.NewClient` 创建的是可复用客户端，内部管理连接池。
- `Addr`、`Password`、`DB` 是最基础配置。
- 启动时可以用 `Ping` 验证 Redis 连接。
- 不要每个请求创建 client，也不要每个请求都 `Ping`。
- Redis 初始化应该封装在清晰的模块里，方便复用和测试。

下一节我们会学习连接池配置：`PoolSize`、`MinIdleConns`，以及超时配置如何影响 Go 服务稳定性。
