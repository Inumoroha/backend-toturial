# 2. zap 日志与请求日志中间件

本节目标：使用 zap 替换简单的 `fmt.Println` 和标准库 `log`，并为 Gin 项目编写一个结构化请求日志中间件。

后端服务上线后，日志就是你排查问题的第一现场。一个接口返回 `500` 时，你不能只知道“出错了”，你还需要知道：

- 哪个请求出错。
- 请求路径是什么。
- HTTP 状态码是多少。
- 请求耗时多久。
- 有没有 request_id 可以串联上下文。
- 错误发生在哪个业务步骤。

这就是结构化日志的价值。

---

## 一、为什么选择 zap

Go 项目中常见日志库有 `logrus`、`zap`、`zerolog`。Gin 后端项目里，zap 很常用，原因是：

- 性能好。
- 支持结构化字段。
- 适合输出 JSON 日志。
- 生态成熟。
- 可以区分开发环境和生产环境配置。

结构化日志不是一整段字符串，而是带字段的日志：

```json
{
  "level": "info",
  "msg": "http request",
  "method": "GET",
  "path": "/api/v1/users",
  "status": 200,
  "latency_ms": 12,
  "request_id": "req-xxx"
}
```

这种日志更适合后续接入日志平台、搜索和聚合。

---

## 二、安装 zap

在项目根目录执行：

```bash
go get go.uber.org/zap
```

验证：

```bash
go list -m go.uber.org/zap
```

---

## 三、封装 logger 初始化

创建 `pkg/logger/logger.go`：

```go
package logger

import (
    "fmt"

    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

type Config struct {
    Level  string
    Format string
}

func New(cfg Config) (*zap.Logger, error) {
    level, err := parseLevel(cfg.Level)
    if err != nil {
        return nil, err
    }

    var zapCfg zap.Config
    if cfg.Format == "console" {
        zapCfg = zap.NewDevelopmentConfig()
    } else {
        zapCfg = zap.NewProductionConfig()
    }

    zapCfg.Level = zap.NewAtomicLevelAt(level)
    zapCfg.Encoding = cfg.Format
    zapCfg.EncoderConfig.TimeKey = "time"
    zapCfg.EncoderConfig.MessageKey = "message"
    zapCfg.EncoderConfig.LevelKey = "level"
    zapCfg.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder

    return zapCfg.Build()
}

func parseLevel(level string) (zapcore.Level, error) {
    var zapLevel zapcore.Level
    if err := zapLevel.UnmarshalText([]byte(level)); err != nil {
        return zapcore.InfoLevel, fmt.Errorf("invalid log level %q: %w", level, err)
    }
    return zapLevel, nil
}
```

这里没有直接在业务代码中到处调用 `zap.NewProduction()`，而是封装成 `logger.New`。这样做的好处是：

- 日志配置集中。
- 测试时可以替换 logger。
- 后续要写入文件或接入日志平台时，改动集中。

---

## 四、在 main 中初始化日志

示例：

```go
log, err := logger.New(logger.Config{
    Level:  cfg.Log.Level,
    Format: cfg.Log.Format,
})
if err != nil {
    panic(err)
}
defer log.Sync()

log.Info("application starting",
    zap.String("app", cfg.App.Name),
    zap.String("env", cfg.App.Env),
    zap.Int("port", cfg.Server.Port),
)
```

注意：

- `defer log.Sync()` 用来刷新缓冲日志。
- Windows 控制台中 `Sync()` 有时会返回无害错误，生产项目可以做兼容处理。
- 不要在每个 handler 中重新创建 logger。

---

## 五、编写 RequestID 中间件

先安装 UUID 库：

```bash
go get github.com/google/uuid
```

创建 `internal/middleware/request_id.go`：

```go
package middleware

import (
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

const RequestIDKey = "request_id"

func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.NewString()
        }

        c.Set(RequestIDKey, requestID)
        c.Header("X-Request-ID", requestID)

        c.Next()
    }
}

func GetRequestID(c *gin.Context) string {
    value, exists := c.Get(RequestIDKey)
    if !exists {
        return ""
    }

    requestID, ok := value.(string)
    if !ok {
        return ""
    }
    return requestID
}
```

RequestID 的作用是给每个请求一个唯一标识。当前端、网关、后端、日志平台都带着这个 ID 时，排查问题会容易很多。

---

## 六、编写请求日志中间件

创建 `internal/middleware/access_log.go`：

```go
package middleware

import (
    "time"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"
)

func AccessLog(log *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        query := c.Request.URL.RawQuery

        c.Next()

        latency := time.Since(start)
        status := c.Writer.Status()
        requestID := GetRequestID(c)

        fields := []zap.Field{
            zap.String("request_id", requestID),
            zap.String("method", c.Request.Method),
            zap.String("path", path),
            zap.String("query", query),
            zap.Int("status", status),
            zap.Int("size", c.Writer.Size()),
            zap.Duration("latency", latency),
            zap.String("client_ip", c.ClientIP()),
        }

        if len(c.Errors) > 0 {
            fields = append(fields, zap.String("errors", c.Errors.String()))
        }

        if status >= 500 {
            log.Error("http request completed", fields...)
            return
        }

        if status >= 400 {
            log.Warn("http request completed", fields...)
            return
        }

        log.Info("http request completed", fields...)
    }
}
```

关键点解释：

- `start := time.Now()`：记录请求开始时间。
- `c.Next()`：放行给后续 handler 执行。
- `time.Since(start)`：计算整个请求耗时。
- `c.Writer.Status()`：读取最终响应状态码。
- `c.Errors`：Gin 上下文中收集到的错误。
- 根据状态码决定日志级别：`2xx/3xx` 为 info，`4xx` 为 warn，`5xx` 为 error。

---

## 七、注册中间件

在路由初始化处：

```go
r := gin.New()

r.Use(middleware.RequestID())
r.Use(middleware.AccessLog(log))
r.Use(gin.Recovery())

r.GET("/ping", func(c *gin.Context) {
    c.JSON(200, gin.H{"message": "pong"})
})
```

中间件顺序建议：

```text
RequestID -> AccessLog -> Recovery -> 业务路由
```

这样请求日志可以拿到 request_id，也能记录经过业务处理后的状态码。

---

## 八、验证请求日志

启动服务：

```bash
go run ./cmd/server
```

请求接口：

```bash
curl -i http://localhost:8080/ping
```

你应该能在响应头中看到：

```text
X-Request-ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

控制台应该能看到类似日志：

```json
{
  "level": "info",
  "time": "2026-07-05T10:00:00.000+0800",
  "message": "http request completed",
  "request_id": "xxx",
  "method": "GET",
  "path": "/ping",
  "status": 200,
  "latency": "1.2ms"
}
```

再请求一个不存在的接口：

```bash
curl -i http://localhost:8080/not-found
```

应该能看到 `404` 请求日志。

---

## 九、业务代码中如何记录日志

在 service 中不要写：

```go
fmt.Println("create user failed")
```

建议写：

```go
log.Error("create user failed",
    zap.String("email", req.Email),
    zap.Error(err),
)
```

日志字段要有价值，但不要泄露敏感信息：

- 可以记录用户 ID。
- 可以记录请求路径。
- 可以记录错误类型。
- 不要记录明文密码。
- 不要记录完整 token。
- 不要记录身份证、银行卡等敏感数据。

---

## 十、常见问题

### 1. 日志没有输出

检查是否注册了中间件：

```go
r.Use(middleware.AccessLog(log))
```

也要检查日志级别。如果配置为 `error`，`info` 日志不会输出。

### 2. 每条请求日志没有 request_id

检查中间件顺序，`RequestID()` 应该在 `AccessLog()` 前面。

### 3. `defer log.Sync()` 报错

某些本地终端可能出现 `invalid argument`。学习阶段可以先忽略，生产代码中可以封装一个 `Sync` 兼容函数。

---

## 十一、练习

请完成：

1. 给请求日志增加 `user_agent` 字段。
2. 给请求日志增加 `referer` 字段。
3. 请求 `/api/v1/users?page=1`，观察 query 字段。
4. 模拟一个返回 `400` 的接口，确认日志级别为 warn。
5. 模拟一个 panic，确认 Recovery 生效且请求日志仍然输出。

---

## 十二、验收标准

本节完成后，请确认：

- 项目已经使用 zap 初始化 logger。
- 请求响应头包含 `X-Request-ID`。
- 每次请求都会输出结构化日志。
- `2xx`、`4xx`、`5xx` 状态码能区分日志级别。
- 日志中没有输出密码、token 等敏感信息。

完成这些后，你就可以进入测试部分了。

