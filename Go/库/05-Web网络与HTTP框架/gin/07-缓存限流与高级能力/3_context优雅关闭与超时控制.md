# 3. context、优雅关闭与超时控制

本节目标：理解 `context.Context` 在 Gin 项目中的作用，并为 HTTP 服务添加超时控制和优雅关闭。

Go 后端项目里，`context` 是非常核心的概念。它不仅仅是一个参数，而是用来传递取消信号、超时时间和请求范围内数据的机制。

---

## 一、为什么需要 context

假设一个请求进入 Gin：

```text
客户端请求
  ↓
handler
  ↓
service
  ↓
repository
  ↓
database / redis
```

如果客户端已经断开连接，或者请求超过超时时间，后面的数据库查询和 Redis 操作就不应该继续无意义地执行。

这时就需要把请求的 context 一路传下去：

```go
func (h *UserHandler) GetProfile(c *gin.Context) {
    user, err := h.userService.GetProfile(c.Request.Context(), userID)
}
```

service：

```go
func (s *UserService) GetProfile(ctx context.Context, userID uint) (*UserDTO, error) {
    return s.repo.FindByID(ctx, userID)
}
```

repository：

```go
func (r *UserRepository) FindByID(ctx context.Context, userID uint) (*model.User, error) {
    var user model.User
    err := r.db.WithContext(ctx).First(&user, userID).Error
    return &user, err
}
```

Redis：

```go
value, err := rdb.Get(ctx, key).Result()
```

---

## 二、context 应该怎么传

推荐规则：

```text
handler 从 c.Request.Context() 取 context
service 方法第一个参数传 ctx
repository 方法第一个参数传 ctx
Redis、GORM、外部 HTTP 调用都使用 ctx
```

示例方法签名：

```go
func (s *UserService) GetByID(ctx context.Context, id uint) (*UserDTO, error)
func (r *UserRepository) FindByID(ctx context.Context, id uint) (*model.User, error)
```

不要在 service 中随便写：

```go
context.Background()
```

这样会丢掉请求取消和超时信号。

---

## 三、请求级超时中间件

可以为请求设置统一超时时间。

创建 `internal/middleware/timeout.go`：

```go
package middleware

import (
    "context"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
)

func Timeout(timeout time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), timeout)
        defer cancel()

        c.Request = c.Request.WithContext(ctx)

        done := make(chan struct{})
        go func() {
            c.Next()
            close(done)
        }()

        select {
        case <-done:
            return
        case <-ctx.Done():
            c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{
                "code":    "REQUEST_TIMEOUT",
                "message": "请求超时",
            })
            return
        }
    }
}
```

这个版本用于学习理解。生产中更推荐谨慎使用成熟中间件或在服务端超时、数据库超时、外部调用超时多层控制。

---

## 四、HTTP Server 超时配置

不要只用：

```go
r.Run(":8080")
```

更推荐显式创建 `http.Server`：

```go
srv := &http.Server{
    Addr:         fmt.Sprintf(":%d", cfg.Server.Port),
    Handler:      r,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}

if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
    log.Fatal("server start failed", zap.Error(err))
}
```

超时字段说明：

- `ReadTimeout`：读取完整请求的最大时间。
- `WriteTimeout`：写响应的最大时间。
- `IdleTimeout`：空闲连接保持时间。

这些配置可以避免慢请求长期占用连接。

---

## 五、优雅关闭是什么

普通关闭可能是：

```text
进程收到退出信号 -> 立即结束 -> 正在处理的请求中断
```

优雅关闭希望做到：

```text
进程收到退出信号
停止接收新请求
等待正在处理的请求完成
超过等待时间则强制退出
```

这对生产部署很重要。发布新版本、容器重启、服务器维护时，都可能触发服务退出。

---

## 六、实现优雅关闭

main 中可以这样写：

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

go func() {
    log.Info("server starting", zap.String("addr", srv.Addr))
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal("server start failed", zap.Error(err))
    }
}()

<-quit
log.Info("server shutting down")

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Error("server forced to shutdown", zap.Error(err))
    return
}

log.Info("server exited")
```

需要导入：

```go
import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)
```

---

## 七、配置化超时时间

可以在配置中增加：

```yaml
server:
  port: 8080
  mode: "debug"
  read_timeout_seconds: 10
  write_timeout_seconds: 10
  idle_timeout_seconds: 60
  shutdown_timeout_seconds: 10
```

结构体：

```go
type ServerConfig struct {
    Port                   int    `mapstructure:"port"`
    Mode                   string `mapstructure:"mode"`
    ReadTimeoutSeconds     int    `mapstructure:"read_timeout_seconds"`
    WriteTimeoutSeconds    int    `mapstructure:"write_timeout_seconds"`
    IdleTimeoutSeconds     int    `mapstructure:"idle_timeout_seconds"`
    ShutdownTimeoutSeconds int    `mapstructure:"shutdown_timeout_seconds"`
}
```

使用：

```go
ReadTimeout: time.Duration(cfg.Server.ReadTimeoutSeconds) * time.Second,
```

---

## 八、常见问题

### 1. 为什么不能到处用 `context.Background()`

因为它不会随着请求取消而取消。请求链路内应该优先使用 `c.Request.Context()`。

### 2. 优雅关闭时 Redis 和数据库要不要关闭

要。HTTP 服务停止后，可以关闭数据库连接池和 Redis 客户端：

```go
sqlDB, _ := db.DB()
_ = sqlDB.Close()
_ = redisClient.Close()
```

### 3. 超时时间越短越好吗

不是。太短会误伤正常请求。应该根据接口类型设置：普通接口几秒内，导出类或批处理接口单独设计。

---

## 九、练习

请完成：

1. 把所有 service 方法第一个参数改为 `context.Context`。
2. GORM 查询统一使用 `db.WithContext(ctx)`。
3. Redis 操作统一传入 ctx。
4. main 中使用 `http.Server` 替代 `r.Run`。
5. 为服务添加 `SIGINT`、`SIGTERM` 优雅关闭。

---

## 十、验收标准

完成后检查：

- handler 没有把 `context.Background()` 传给 service。
- repository 查询使用了 `WithContext(ctx)`。
- Redis 操作使用请求 ctx。
- 服务配置了 ReadTimeout 和 WriteTimeout。
- Ctrl+C 关闭服务时有 shutdown 日志。

做到这些后，项目就具备更像生产服务的生命周期管理能力。

