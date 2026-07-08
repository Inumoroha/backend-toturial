# 4. 请求耗时与 Request ID 中间件

本节目标：给每个请求生成唯一的 Request ID，并在日志和响应头中携带它。同时记录请求耗时，为后续排查问题打基础。

---

## 一、为什么需要 Request ID

真实项目中，一个请求可能会产生多条日志：

```text
请求进入日志
参数校验日志
数据库查询日志
错误日志
请求结束日志
```

如果没有统一标识，很难判断这些日志是不是同一次请求产生的。

Request ID 的作用就是：

```text
给每一次请求一个唯一编号。
```

例如：

```text
X-Request-ID: 8d4f9d32-2a8a-4d8d-9a1f-0f6c3d84f1a2
```

---

## 二、安装 uuid 库

```bash
go get github.com/google/uuid
```

这个库用来生成唯一 ID。

---

## 三、实现 RequestID 中间件

创建：

```text
internal/middleware/request_id.go
```

代码：

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
```

这里有两个细节：

```text
如果客户端已经传了 X-Request-ID，就沿用它。
如果没有传，就由服务端生成。
```

这样方便网关、前端或其他服务传递链路 ID。

---

## 四、在 handler 中读取 Request ID

```go
requestID, _ := c.Get(middleware.RequestIDKey)
```

或：

```go
requestID := c.GetString(middleware.RequestIDKey)
```

一般业务 handler 不需要主动读取，但日志和错误处理会用到。

---

## 五、实现请求耗时日志

创建：

```text
internal/middleware/access_log.go
```

代码：

```go
package middleware

import (
    "log"
    "time"

    "github.com/gin-gonic/gin"
)

func AccessLog() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        c.Next()

        requestID := c.GetString(RequestIDKey)
        latency := time.Since(start)

        log.Printf(
            "request_id=%s method=%s path=%s status=%d latency=%s client_ip=%s",
            requestID,
            c.Request.Method,
            c.Request.URL.Path,
            c.Writer.Status(),
            latency,
            c.ClientIP(),
        )
    }
}
```

日志示例：

```text
request_id=8d4f... method=GET path=/api/v1/profile status=200 latency=2.1ms client_ip=127.0.0.1
```

---

## 六、注册顺序

推荐：

```go
r := gin.New()

r.Use(middleware.RequestID())
r.Use(middleware.AccessLog())
r.Use(middleware.Recovery())
```

为什么 RequestID 在 AccessLog 前面？

因为 AccessLog 需要读取 request_id。如果顺序反过来，日志中可能拿不到 request_id。

---

## 七、响应头验证

启动服务后请求：

```http
GET http://localhost:8080/ping
```

查看响应头，应该能看到：

```text
X-Request-ID: ...
```

如果你使用 `requests.http`，响应面板里可以查看 Headers。

---

## 八、客户端传入 Request ID

请求：

```http
GET http://localhost:8080/ping
X-Request-ID: demo-request-id
```

期望响应头：

```text
X-Request-ID: demo-request-id
```

这样可以验证服务端确实沿用了客户端传入的 ID。

---

## 九、结合 zap 的版本

后续工程化阶段会使用 zap。这里先给一个方向：

```go
logger.Info("http request",
    zap.String("request_id", requestID),
    zap.String("method", c.Request.Method),
    zap.String("path", c.Request.URL.Path),
    zap.Int("status", c.Writer.Status()),
    zap.Duration("latency", latency),
)
```

学习阶段先用标准库 `log.Printf` 即可。

---

## 十、常见问题

### 1. 日志里 request_id 为空

检查中间件注册顺序：

```go
r.Use(RequestID())
r.Use(AccessLog())
```

不要反过来。

### 2. 响应头没有 X-Request-ID

检查 RequestID 中间件是否注册到了当前路由。

如果只挂在某个路由组上，组外路由不会有这个 header。

### 3. 每个请求的 request_id 都一样

检查是否把 requestID 定义成了全局变量。应该每次请求在中间件内部生成。

---

## 十一、本节练习

完成：

1. 安装 `github.com/google/uuid`。
2. 实现 `RequestID()` 中间件。
3. 实现 `AccessLog()` 中间件。
4. 确认响应头有 `X-Request-ID`。
5. 确认日志中包含 request_id、method、path、status、latency。
6. 测试客户端传入 `X-Request-ID` 时服务端是否沿用。

---

## 十二、本节验收

你应该能够回答：

- Request ID 解决什么问题？
- 为什么 RequestID 中间件要注册在 AccessLog 前面？
- `c.Set` 和 `c.GetString` 可以用来做什么？
- 请求耗时为什么要在 `c.Next()` 之后计算？
- 响应头中的 `X-Request-ID` 对排查问题有什么帮助？

