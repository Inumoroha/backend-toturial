# 7. HTTP Server 完整项目：路由、中间件、超时与优雅关闭

本节目标：从零写一个可运行的 Go HTTP 服务，完整练习路由注册、中间件、请求日志、panic 恢复、超时配置、健康检查和优雅关闭。

这一节不是讲 `net/http` 的 API 清单，而是把后端项目里最常见的一套 HTTP Server 骨架搭出来。

---

## 一、最终目标

完成后你会得到一个项目：

```text
http-server-lab/
  go.mod
  cmd/server/main.go
  internal/httpx/middleware.go
  internal/httpx/response.go
  internal/user/handler.go
```

它提供：

```text
GET  /healthz
GET  /users/{id}
POST /users
```

同时具备：

- 服务端读超时、写超时、空闲连接超时。
- 每个请求输出访问日志。
- panic 后返回 `500`，服务不中断。
- 收到 `Ctrl+C` 后停止接收新请求，并等待正在处理的请求结束。

---

## 二、创建项目目录

```powershell
mkdir http-server-lab
cd http-server-lab
go mod init example.com/http-server-lab
mkdir cmd
mkdir cmd\server
mkdir internal
mkdir internal\httpx
mkdir internal\user
```

这些目录的职责是：

```text
cmd/server          程序入口，只负责组装依赖和启动服务。
internal/httpx      HTTP 通用工具，例如中间件、JSON 响应。
internal/user       用户相关 handler，后续可继续拆 service/repository。
```

初学者容易把所有代码都写进 `main.go`，这样能跑，但后续一加接口就会变乱。本节故意提前做一个小分层。

---

## 三、实现 JSON 响应工具

创建文件：

```text
internal/httpx/response.go
```

写入：

```go
package httpx

import (
    "encoding/json"
    "net/http"
)

func WriteJSON(w http.ResponseWriter, status int, value any) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(value)
}

func WriteError(w http.ResponseWriter, status int, message string) {
    WriteJSON(w, status, map[string]string{
        "error": message,
    })
}
```

逐行理解：

- `Content-Type` 告诉客户端响应体是 JSON。
- `WriteHeader(status)` 必须在写响应体之前调用。
- `json.NewEncoder(w).Encode(value)` 直接把结构体编码到响应流。
- `_ = ...` 表示这里故意忽略编码错误，因为对 `http.ResponseWriter` 写失败通常说明客户端已经断开。

后端项目不要在每个 handler 里重复写 JSON 响应格式，统一函数会让错误处理更稳定。

---

## 四、实现日志与恢复中间件

创建文件：

```text
internal/httpx/middleware.go
```

写入：

```go
package httpx

import (
    "log"
    "net/http"
    "runtime/debug"
    "time"
)

type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func RequestLog(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s remote=%s cost=%s",
            r.Method,
            r.URL.Path,
            r.RemoteAddr,
            time.Since(start),
        )
    })
}

func Recover(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if v := recover(); v != nil {
                log.Printf("panic: %v\n%s", v, debug.Stack())
                WriteError(w, http.StatusInternalServerError, "internal server error")
            }
        }()

        next.ServeHTTP(w, r)
    })
}
```

这里有几个关键点：

- 中间件本质是接收一个 `http.Handler`，返回一个新的 `http.Handler`。
- `Chain` 从后往前包裹，保证调用顺序符合书写顺序。
- `RequestLog` 记录请求方法、路径、客户端地址和耗时。
- `Recover` 用 `defer + recover` 捕获 panic，避免整个进程因为一个请求崩掉。

请求进入后的顺序是：

```text
Recover -> RequestLog -> mux -> handler
```

如果 handler panic，`Recover` 会捕获。日志中间件仍然能记录这个请求的耗时。

---

## 五、实现用户 Handler

创建文件：

```text
internal/user/handler.go
```

写入：

```go
package user

import (
    "encoding/json"
    "net/http"
    "strings"

    "example.com/http-server-lab/internal/httpx"
)

type Handler struct{}

func NewHandler() *Handler {
    return &Handler{}
}

type createUserRequest struct {
    Name string `json:"name"`
}

type userResponse struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}

func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
    mux.HandleFunc("GET /users/{id}", h.getUser)
    mux.HandleFunc("POST /users", h.createUser)
}

func (h *Handler) getUser(w http.ResponseWriter, r *http.Request) {
    id := strings.TrimSpace(r.PathValue("id"))
    if id == "" {
        httpx.WriteError(w, http.StatusBadRequest, "missing user id")
        return
    }

    httpx.WriteJSON(w, http.StatusOK, userResponse{
        ID:   id,
        Name: "demo-user-" + id,
    })
}

func (h *Handler) createUser(w http.ResponseWriter, r *http.Request) {
    defer r.Body.Close()

    var req createUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        httpx.WriteError(w, http.StatusBadRequest, "invalid json body")
        return
    }

    req.Name = strings.TrimSpace(req.Name)
    if req.Name == "" {
        httpx.WriteError(w, http.StatusBadRequest, "name is required")
        return
    }

    httpx.WriteJSON(w, http.StatusCreated, userResponse{
        ID:   "u_10001",
        Name: req.Name,
    })
}
```

逐段理解：

- `RegisterRoutes` 把路由注册动作放在 handler 内部，`main.go` 不需要知道每个接口函数名。
- `GET /users/{id}` 是 Go 1.22 标准库 `ServeMux` 的新写法。
- `r.PathValue("id")` 从路径参数里取值。
- `defer r.Body.Close()` 用于释放请求体资源。
- JSON 解析失败返回 `400`，不是 `500`，因为这是客户端请求格式错误。
- 创建成功返回 `201 Created`。

如果你的 Go 版本低于 1.22，需要把路由写成旧风格，或者升级 Go。

---

## 六、实现 main.go

创建文件：

```text
cmd/server/main.go
```

写入：

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "example.com/http-server-lab/internal/httpx"
    "example.com/http-server-lab/internal/user"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
        httpx.WriteJSON(w, http.StatusOK, map[string]string{
            "status": "ok",
        })
    })

    userHandler := user.NewHandler()
    userHandler.RegisterRoutes(mux)

    handler := httpx.Chain(
        mux,
        httpx.Recover,
        httpx.RequestLog,
    )

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           handler,
        ReadHeaderTimeout: 3 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    go func() {
        log.Printf("listening on %s", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen failed: %v", err)
        }
    }()

    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
    <-stop

    log.Println("shutting down server")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("graceful shutdown failed: %v", err)
        _ = srv.Close()
    }

    log.Println("server stopped")
}
```

这里是本节最重要的部分：

- `ReadHeaderTimeout` 防止客户端慢慢发送请求头，占住连接。
- `ReadTimeout` 限制读取完整请求的时间。
- `WriteTimeout` 限制服务端写响应的时间。
- `IdleTimeout` 限制 keep-alive 空闲连接保留多久。
- `ListenAndServe` 放到 goroutine 中运行，主 goroutine 等待退出信号。
- `Shutdown(ctx)` 会先关闭监听 socket，不再接收新连接，然后等待已有请求结束。
- 如果 `Shutdown` 超时失败，再调用 `Close` 强制关闭。

一个线上 Go HTTP 服务不应该只写：

```go
http.ListenAndServe(":8080", nil)
```

这个写法适合临时 demo，不适合长期运行的后端服务。

---

## 七、启动服务

执行：

```powershell
go run ./cmd/server
```

预期看到：

```text
listening on :8080
```

健康检查：

```powershell
curl.exe -i http://localhost:8080/healthz
```

预期：

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

响应体：

```json
{"status":"ok"}
```

查询用户：

```powershell
curl.exe -i http://localhost:8080/users/42
```

创建用户：

```powershell
curl.exe -i -X POST http://localhost:8080/users `
  -H "Content-Type: application/json" `
  -d "{\"name\":\"Alice\"}"
```

预期返回 `201 Created`。

---

## 八、观察请求日志

每访问一次接口，服务端会输出类似日志：

```text
GET /healthz remote=[::1]:53120 cost=0s
GET /users/42 remote=[::1]:53121 cost=0s
POST /users remote=[::1]:53122 cost=1.2ms
```

这些字段在排障时非常有用：

- `method` 告诉你接口语义。
- `path` 告诉你访问了哪个资源。
- `remote` 可帮助判断请求来自本机、容器还是其他机器。
- `cost` 是最基础的耗时指标。

生产环境一般还会记录状态码、请求 ID、User-Agent、上游耗时等。本节先把最小骨架跑通。

---

## 九、验证优雅关闭

启动服务后按：

```text
Ctrl+C
```

预期日志：

```text
shutting down server
server stopped
```

如果你想更明显地观察优雅关闭，可以临时在 `getUser` 中加入：

```go
time.Sleep(5 * time.Second)
```

然后发起请求，在请求未结束时按 `Ctrl+C`。如果 `Shutdown` 生效，服务会等待这个请求处理完成，或者等到 10 秒超时。

注意：加了 `time.Sleep` 需要补 import，实验结束后要删除。

---

## 十、常见问题

### 1. 为什么要设置这么多 timeout？

因为真实网络里客户端可能很慢，也可能恶意占住连接。

如果没有 `ReadHeaderTimeout`，慢速客户端可以一点点发送请求头，让服务端长时间保留连接。连接多了以后，文件描述符和 goroutine 都会被耗尽。

### 2. `Shutdown` 和 `Close` 有什么区别？

`Shutdown` 是温和关闭：

```text
停止接收新连接。
等待已有请求处理完。
超过上下文时间后返回错误。
```

`Close` 是强制关闭：

```text
直接关闭监听器和连接。
正在处理的请求可能失败。
```

所以通常先 `Shutdown`，失败后再 `Close`。

### 3. 为什么 handler 里要 `defer r.Body.Close()`？

标准库 server 侧通常会负责关闭请求体，但在业务代码里显式关闭能表达资源释放意识。更重要的是，客户端代码里必须关闭响应体，否则连接池无法复用。

### 4. panic 之后还能继续服务吗？

如果 panic 被 `Recover` 中间件捕获，本次请求返回 `500`，进程继续运行。

如果没有 recover，一个 handler 中的 panic 会被 net/http 内部恢复并关闭该连接，但你很难统一记录结构化错误。自己加恢复中间件更适合项目管理。

### 5. 为什么不用框架？

学习阶段先用标准库能看清楚 HTTP 服务的基本结构。等你理解路由、中间件、超时和关闭之后，再使用 Gin、Echo、Chi 会更稳。

---

## 十一、练习

请在当前项目上继续完成：

1. 给日志中间件增加状态码记录。
2. 增加 `GET /panic`，故意 panic，验证恢复中间件。
3. 把监听地址从 `:8080` 改成从环境变量 `HTTP_ADDR` 读取。
4. 给 `POST /users` 增加最大请求体限制，例如 1MB。

这些练习都很接近真实后端项目。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 从零创建一个 Go HTTP Server 项目。
- 使用 `http.ServeMux` 注册带方法和路径参数的路由。
- 编写 JSON 响应工具函数。
- 编写请求日志中间件。
- 编写 panic 恢复中间件。
- 为 `http.Server` 配置关键超时。
- 使用 `signal.Notify` 接收退出信号。
- 使用 `Shutdown` 实现优雅关闭。
- 用 `curl` 验证接口行为。
- 解释为什么线上服务不能只写 `http.ListenAndServe(":8080", nil)`。

