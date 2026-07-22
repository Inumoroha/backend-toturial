# 4. Redis 初始化与连接检查

本节目标：在 Gin 项目中初始化 Redis 客户端，完成配置读取、连接检查和关闭资源。

Redis 在项目中通常是一个基础设施依赖，应该在程序启动时初始化，然后通过依赖注入传给需要使用的 service 或 cache 组件。

---

## 一、安装 go-redis

执行：

```bash
go get github.com/redis/go-redis/v9
```

验证：

```bash
go list -m github.com/redis/go-redis/v9
```

---

## 二、准备 Redis 配置

在 `config/config.yaml` 中增加：

```yaml
redis:
  addr: "127.0.0.1:6379"
  password: ""
  db: 0
  dial_timeout_seconds: 5
  read_timeout_seconds: 3
  write_timeout_seconds: 3
```

配置结构体：

```go
type RedisConfig struct {
    Addr                string `mapstructure:"addr"`
    Password            string `mapstructure:"password"`
    DB                  int    `mapstructure:"db"`
    DialTimeoutSeconds  int    `mapstructure:"dial_timeout_seconds"`
    ReadTimeoutSeconds  int    `mapstructure:"read_timeout_seconds"`
    WriteTimeoutSeconds int    `mapstructure:"write_timeout_seconds"`
}
```

总配置中增加：

```go
type Config struct {
    App      AppConfig      `mapstructure:"app"`
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    Redis    RedisConfig    `mapstructure:"redis"`
    JWT      JWTConfig      `mapstructure:"jwt"`
    Log      LogConfig      `mapstructure:"log"`
}
```

---

## 三、启动 Redis

如果本机没有 Redis，可以用 Docker：

```bash
docker run --name redis-dev -p 6379:6379 -d redis:7
```

查看容器：

```bash
docker ps
```

验证：

```bash
docker exec -it redis-dev redis-cli ping
```

返回：

```text
PONG
```

---

## 四、封装 Redis 初始化

创建 `internal/cache/redis.go`：

```go
package cache

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"

    "gin-user-api/internal/config"
)

func NewRedisClient(ctx context.Context, cfg config.RedisConfig) (*redis.Client, error) {
    client := redis.NewClient(&redis.Options{
        Addr:         cfg.Addr,
        Password:     cfg.Password,
        DB:           cfg.DB,
        DialTimeout:  time.Duration(cfg.DialTimeoutSeconds) * time.Second,
        ReadTimeout:  time.Duration(cfg.ReadTimeoutSeconds) * time.Second,
        WriteTimeout: time.Duration(cfg.WriteTimeoutSeconds) * time.Second,
    })

    if err := client.Ping(ctx).Err(); err != nil {
        _ = client.Close()
        return nil, fmt.Errorf("ping redis: %w", err)
    }

    return client, nil
}
```

关键点解释：

- `redis.NewClient` 只是创建客户端，不代表连接一定可用。
- `Ping(ctx)` 用来启动时验证连接。
- 如果 ping 失败，要关闭 client。
- 返回错误时带上上下文信息，方便排查。

---

## 五、在 main 中初始化

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

redisClient, err := cache.NewRedisClient(ctx, cfg.Redis)
if err != nil {
    logger.Fatal("connect redis failed", zap.Error(err))
}
defer redisClient.Close()

logger.Info("redis connected", zap.String("addr", cfg.Redis.Addr))
```

如果 Redis 是项目必需依赖，连接失败可以直接启动失败。如果只是缓存依赖，也可以允许降级，但初学阶段建议先失败得清楚。

---

## 六、写一个简单连接测试接口

学习阶段可以临时加一个健康检查：

```go
r.GET("/health/redis", func(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), time.Second)
    defer cancel()

    if err := redisClient.Ping(ctx).Err(); err != nil {
        c.JSON(http.StatusServiceUnavailable, gin.H{
            "status": "down",
            "error":  err.Error(),
        })
        return
    }

    c.JSON(http.StatusOK, gin.H{"status": "up"})
})
```

访问：

```bash
curl http://localhost:8080/health/redis
```

返回：

```json
{"status":"up"}
```

说明应用可以访问 Redis。

---

## 七、Redis 客户端如何传递

不要在业务代码中到处创建 Redis 客户端：

```go
redis.NewClient(...)
```

推荐在 main 中创建一次，然后注入：

```go
userCache := cache.NewUserCache(redisClient)
userService := service.NewUserService(userRepo, userCache)
userHandler := handler.NewUserHandler(userService)
```

这样生命周期清晰，也方便测试时替换 fake cache。

---

## 八、常见问题

### 1. 连接 Redis 失败

检查：

```bash
docker ps
docker logs redis-dev
docker exec -it redis-dev redis-cli ping
```

再检查配置中的地址是否是：

```text
127.0.0.1:6379
```

### 2. Docker 中连接本机 Redis 失败

如果应用也跑在容器中，`127.0.0.1` 指的是应用容器自身，不是宿主机。Docker Compose 中通常用服务名：

```yaml
redis:
  image: redis:7

app:
  environment:
    APP_REDIS_ADDR: redis:6379
```

### 3. Redis 密码错误

如果 Redis 没有设置密码，配置中 password 应为空字符串。设置了密码则必须一致。

---

## 九、练习

请完成：

1. 在配置中增加 Redis 配置。
2. 编写 `NewRedisClient`。
3. 启动时 ping Redis。
4. 添加 `/health/redis` 临时接口。
5. 关闭服务时调用 `redisClient.Close()`。

---

## 十、验收标准

运行：

```bash
go run ./cmd/server
curl http://localhost:8080/health/redis
```

确认：

- Redis 容器正常运行。
- 应用启动时输出 Redis 连接成功日志。
- `/health/redis` 返回 `up`。
- Redis 停止后应用能输出清晰错误。

完成 Redis 初始化后，就可以实现用户详情缓存。

