# 2. CORS、日志与 Recovery

本节目标：掌握 Gin 项目中最常见的三个基础中间件：请求日志、panic recovery 和 CORS。它们通常在项目很早期就应该加入，因为它们直接影响调试、稳定性和前后端联调。

---

## 一、为什么先学这三个中间件

这三个中间件解决的问题非常基础：

```text
请求日志：我怎么知道请求有没有进来、返回了什么状态码？
Recovery：某个 handler panic 时，服务会不会直接崩掉？
CORS：前端浏览器为什么访问不了后端接口？
```

Gin 的 `gin.Default()` 已经默认包含：

```text
Logger
Recovery
```

但真实项目中，你迟早会自定义它们。

---

## 二、请求日志中间件

先写一个简单版本：

```go
func AccessLog() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        c.Next()

        latency := time.Since(start)
        log.Printf(
            "%s %s %d %s",
            c.Request.Method,
            c.Request.URL.Path,
            c.Writer.Status(),
            latency,
        )
    }
}
```

需要导入：

```go
import (
    "log"
    "time"

    "github.com/gin-gonic/gin"
)
```

注册：

```go
r := gin.New()
r.Use(AccessLog())
r.Use(gin.Recovery())
```

访问：

```http
GET http://localhost:8080/ping
```

日志类似：

```text
GET /ping 200 1.2ms
```

---

## 三、为什么耗时统计要写在 c.Next() 后

中间件中：

```go
start := time.Now()
c.Next()
latency := time.Since(start)
```

`c.Next()` 会执行后续中间件和 handler。只有等它执行完，才能知道整个请求耗时。

如果你在 `c.Next()` 前计算耗时，只能得到几乎为 0 的时间。

---

## 四、日志应该包含哪些字段

学习阶段至少包含：

```text
method
path
status
latency
```

工程化阶段建议增加：

```text
request_id
client_ip
user_agent
error
user_id
```

这些字段后面排查问题会非常有用。

---

## 五、Recovery 是什么

panic 示例：

```go
r.GET("/panic", func(c *gin.Context) {
    panic("something wrong")
})
```

如果没有 recovery，中小项目里一个 panic 可能导致进程退出。

使用 Gin 自带 recovery：

```go
r.Use(gin.Recovery())
```

请求 `/panic` 时，Gin 会返回 500，并把错误栈打印到终端。

---

## 六、自定义 Recovery 响应

为了保持统一响应，可以自定义：

```go
func Recovery() gin.HandlerFunc {
    return gin.CustomRecovery(func(c *gin.Context, recovered interface{}) {
        log.Printf("panic recovered: %v", recovered)
        response.Fail(c, http.StatusInternalServerError, response.CodeInternalError, "internal server error")
    })
}
```

注册：

```go
r := gin.New()
r.Use(AccessLog())
r.Use(Recovery())
```

这样 panic 时也能返回统一 JSON：

```json
{
  "code": 50001,
  "message": "internal server error"
}
```

---

## 七、CORS 是什么

CORS 是浏览器的跨域安全机制。

例如前端运行在：

```text
http://localhost:5173
```

后端运行在：

```text
http://localhost:8080
```

端口不同，也属于跨域。

浏览器会拦截不符合 CORS 规则的请求。

注意：CORS 是浏览器限制，不是 Postman 或 curl 的限制。所以 Postman 能通，不代表浏览器也能通。

---

## 八、安装 CORS 中间件

```bash
go get github.com/gin-contrib/cors
```

配置：

```go
import "github.com/gin-contrib/cors"

r.Use(cors.New(cors.Config{
    AllowOrigins: []string{"http://localhost:5173"},
    AllowMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
    AllowHeaders: []string{"Origin", "Content-Type", "Authorization"},
}))
```

字段解释：

```text
AllowOrigins
允许哪些前端源访问。

AllowMethods
允许哪些 HTTP 方法。

AllowHeaders
允许请求携带哪些 Header。
```

如果前端要传 JWT，必须允许：

```text
Authorization
```

---

## 九、开发环境和生产环境的 CORS

开发环境可以允许：

```go
AllowOrigins: []string{"http://localhost:5173"}
```

不要在生产环境随意使用：

```go
AllowAllOrigins: true
```

原因是生产环境应该明确允许可信前端域名，例如：

```text
https://app.example.com
```

---

## 十、推荐中间件注册顺序

一个基础顺序：

```go
r := gin.New()

r.Use(RequestID())
r.Use(AccessLog())
r.Use(Recovery())
r.Use(cors.New(cors.Config{
    AllowOrigins: []string{"http://localhost:5173"},
    AllowMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
    AllowHeaders: []string{"Origin", "Content-Type", "Authorization"},
}))
```

说明：

- RequestID 越早越好，后续日志可以使用。
- AccessLog 包住后续 handler，统计完整耗时。
- Recovery 捕获后续 panic。
- CORS 处理浏览器跨域。

实际项目中顺序可以根据团队规范调整，但要理解每个中间件的位置影响什么。

---

## 十一、常见问题

### 1. Postman 正常，浏览器报跨域

这是正常现象。CORS 是浏览器机制。

检查后端是否允许了前端 origin。

### 2. 前端带 Authorization 失败

检查 CORS 配置中是否包含：

```go
AllowHeaders: []string{"Authorization"}
```

### 3. panic 返回的不是统一格式

检查是否使用自定义 Recovery，而不是只用 `gin.Recovery()`。

### 4. 日志没有打印请求

检查是否注册了日志中间件：

```go
r.Use(AccessLog())
```

---

## 十二、本节练习

完成：

1. 把 `gin.Default()` 改为 `gin.New()`。
2. 手动注册 `AccessLog()`。
3. 手动注册自定义 `Recovery()`。
4. 添加 `/panic` 路由测试 panic 响应。
5. 安装并配置 `gin-contrib/cors`。
6. 确认 CORS 允许 `Authorization` header。

---

## 十三、本节验收

你应该能够回答：

- 请求日志为什么要在 `c.Next()` 后计算耗时？
- Recovery 解决什么问题？
- 为什么不要把 panic 细节直接返回给用户？
- CORS 是后端限制还是浏览器限制？
- 前端带 JWT 时 CORS 必须允许哪个 Header？
- 开发环境和生产环境 CORS 配置有什么区别？


