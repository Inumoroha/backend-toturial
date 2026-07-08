# Go net/http 系统学习路线图

> 目标：从能写简单 HTTP 服务，逐步成长为能设计、调试、优化和维护生产级 Go Web 后端服务的工程师。

## 0. 学习前置要求

在系统学习 `net/http` 前，建议先具备这些 Go 基础：

- 熟悉 `package`、`func`、`struct`、`interface`、`method`
- 理解 `error` 处理方式
- 会使用 `go mod`
- 理解 goroutine、channel、context 的基本概念
- 会写基础单元测试：`testing`、`go test`

推荐先完成的小练习：

- 写一个命令行程序，读取参数并输出结果
- 写一个简单结构体，并为它实现接口
- 用 goroutine 并发执行 3 个任务，并收集结果
- 使用 `context.WithTimeout` 控制任务超时

## 1. HTTP 与 Web 基础

### 学习目标

先理解 HTTP 协议本身，再学习 Go 的封装。这样后面遇到请求头、状态码、连接复用、超时、代理、Cookie 等问题时，不会只停留在 API 表面。

### 必学内容

- URL、Path、Query、Fragment
- HTTP Method：`GET`、`POST`、`PUT`、`PATCH`、`DELETE`
- HTTP Status Code：`2xx`、`3xx`、`4xx`、`5xx`
- Header、Body、Cookie
- Content-Type 与 JSON
- HTTP/1.1 的 keep-alive
- HTTPS 与 TLS 的基本概念

### 推荐练习

用 curl 或 Postman 练习：

```bash
curl -i http://localhost:8080/hello
curl -X POST http://localhost:8080/users -H "Content-Type: application/json" -d "{\"name\":\"alice\"}"
curl -I http://localhost:8080/health
```

### 阶段产出

写一份自己的 HTTP 笔记，至少解释清楚：

- GET 和 POST 的差异
- 400、401、403、404、500 的含义
- Header 和 Body 分别适合放什么
- 为什么服务端需要设置超时时间

## 2. net/http 入门：写出第一个 HTTP 服务

### 学习目标

掌握 `net/http` 服务端最基础的模型：`Handler`、`ServeMux`、`ResponseWriter`、`Request`。

### 必学 API

- `http.ListenAndServe`
- `http.HandleFunc`
- `http.Handler`
- `http.HandlerFunc`
- `http.ServeMux`
- `http.ResponseWriter`
- `http.Request`

### 示例方向

实现一个最小服务：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "hello, net/http")
	})

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### 推荐练习

- `/hello`：返回纯文本
- `/health`：返回 `200 OK`
- `/users/{id}`：读取路径参数
- `/search?q=go`：读取 Query 参数
- 访问不存在的路径时返回 `404`

### 阶段产出

完成一个基础服务，包含：

- 至少 4 个路由
- 能读取 Query 参数
- 能读取 Path 参数
- 能返回不同状态码

## 3. 请求与响应处理

### 学习目标

掌握真实业务中最常见的输入输出处理：JSON、表单、Header、Cookie、状态码。

### 必学内容

- 读取请求体：`io.ReadAll`、`json.NewDecoder`
- 写 JSON 响应：`json.NewEncoder`
- 设置响应头：`w.Header().Set`
- 设置状态码：`w.WriteHeader`
- 读取 Header：`r.Header.Get`
- 读取 Cookie：`r.Cookie`
- 设置 Cookie：`http.SetCookie`
- 限制请求体大小：`http.MaxBytesReader`

### 推荐封装

建议自己写两个小工具函数：

```go
func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, message string) {
	writeJSON(w, status, map[string]string{"error": message})
}
```

### 推荐练习

实现一个内存版 Todo API：

- `GET /todos`：获取 Todo 列表
- `POST /todos`：创建 Todo
- `GET /todos/{id}`：获取单个 Todo
- `PATCH /todos/{id}`：更新 Todo
- `DELETE /todos/{id}`：删除 Todo

### 阶段产出

完成一个不依赖数据库的 JSON API 服务，要求：

- 请求参数校验
- 统一 JSON 响应格式
- 统一错误响应格式
- 正确使用状态码

## 4. Handler、Middleware 与路由组织

### 学习目标

理解 Go Web 服务的核心组合方式：一切围绕 `http.Handler` 组合。

### 必学内容

- `http.Handler` 接口：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

- `http.HandlerFunc`
- 中间件的本质：接收一个 `http.Handler`，返回一个新的 `http.Handler`
- 日志中间件
- Panic Recovery 中间件
- CORS 中间件
- 鉴权中间件

### 中间件模板

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Printf("%s %s", r.Method, r.URL.Path)
		next.ServeHTTP(w, r)
	})
}
```

### 推荐练习

给 Todo API 加上：

- 请求日志
- Panic 恢复
- 简单 Token 鉴权
- 请求耗时统计
- Request ID

### 阶段产出

形成自己的基础 Web 工程结构：

```text
.
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handler/
│   ├── middleware/
│   ├── model/
│   └── service/
└── go.mod
```

## 5. Server 配置与生产级基础

### 学习目标

不要只会 `http.ListenAndServe`。生产环境中更常用的是显式创建 `http.Server`，并配置超时、关闭逻辑和错误处理。

### 必学内容

- `http.Server`
- `ReadTimeout`
- `ReadHeaderTimeout`
- `WriteTimeout`
- `IdleTimeout`
- `MaxHeaderBytes`
- 优雅关闭：`Server.Shutdown`
- 信号处理：`os/signal`
- `context.Context`

### 推荐写法

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           mux,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

### 推荐练习

- 服务收到 `Ctrl+C` 后优雅退出
- 正在处理的请求最多等待 5 秒
- 超时后强制退出
- 模拟慢请求，观察超时行为

### 阶段产出

把 Todo API 改造成具备生产级基础的服务：

- 明确配置 `http.Server`
- 支持优雅关闭
- 配置合理超时
- 日志中记录启动、关闭和错误信息

## 6. Context 深入

### 学习目标

掌握请求生命周期控制。Go 后端工程里，`context.Context` 是贯穿 HTTP、数据库、RPC、缓存调用的核心工具。

### 必学内容

- `r.Context()`
- `context.WithTimeout`
- `context.WithCancel`
- 请求取消传播
- 在 handler、service、repository 间传递 context
- 不滥用 `context.Value`

### 推荐练习

- 写一个 `/slow` 接口，模拟耗时任务
- 客户端断开连接后，服务端及时取消任务
- 给下游模拟调用设置 2 秒超时
- 日志中打印 context 取消原因

### 阶段产出

能解释并演示：

- 为什么 handler 里应该优先使用 `r.Context()`
- 客户端断开连接后服务端会发生什么
- 超时和取消有什么区别

## 7. 测试 net/http 服务

### 学习目标

学会测试 Handler，而不是每次都手动启动服务。

### 必学内容

- `net/http/httptest`
- `httptest.NewRequest`
- `httptest.NewRecorder`
- `httptest.NewServer`
- 表格驱动测试
- JSON 响应断言
- Mock service 层

### 推荐练习

为 Todo API 编写测试：

- 创建 Todo 成功
- JSON 参数错误返回 `400`
- 查询不存在的 Todo 返回 `404`
- 未授权请求返回 `401`
- Panic 被 recovery 中间件捕获并返回 `500`

### 阶段产出

测试覆盖这些层次：

- Handler 单元测试
- Middleware 单元测试
- 基于 `httptest.NewServer` 的集成测试

## 8. HTTP Client

### 学习目标

后端工程师不仅写服务端，也经常调用第三方 API。要掌握 `net/http` 客户端的正确用法。

### 必学内容

- `http.Client`
- `http.NewRequestWithContext`
- `client.Do`
- 读取响应体并关闭：`defer resp.Body.Close()`
- 设置 Header
- 处理非 `2xx` 状态码
- 配置超时
- 连接池与 `http.Transport`

### 推荐练习

写一个 GitHub 用户信息查询客户端：

- 输入用户名
- 请求 GitHub API
- 解析 JSON 响应
- 处理超时
- 处理 `404`
- 写单元测试，使用 `httptest.NewServer` 模拟 GitHub API

### 阶段产出

封装一个可靠的 HTTP Client：

- 所有请求都带 context
- 有全局超时
- 正确关闭响应体
- 对错误状态码有统一处理

## 9. 文件上传、下载与静态文件

### 学习目标

掌握常见 Web 后端文件处理能力。

### 必学内容

- `r.ParseMultipartForm`
- `r.FormFile`
- `http.MaxBytesReader`
- `http.ServeFile`
- `http.FileServer`
- `http.ServeContent`
- 下载响应头：`Content-Disposition`

### 推荐练习

- 上传单个图片文件
- 限制文件大小
- 校验文件后缀和 Content-Type
- 下载指定文件
- 提供静态文件目录访问

### 阶段产出

实现一个简单文件服务：

- `POST /upload`
- `GET /files/{name}`
- `GET /static/*`

## 10. 安全基础

### 学习目标

理解 Web 服务的常见安全风险，并知道 `net/http` 中对应的处理方式。

### 必学内容

- 请求体大小限制
- 超时配置
- CORS
- Cookie 安全属性：`HttpOnly`、`Secure`、`SameSite`
- 基础鉴权：Bearer Token
- 防止敏感错误信息暴露
- TLS 基础
- 反向代理后的真实 IP 问题

### 推荐练习

给 Todo API 加安全增强：

- 所有写操作要求 Token
- 请求体最大 1MB
- Cookie 设置 `HttpOnly`
- 错误响应不暴露内部堆栈
- 配置 CORS 白名单

### 阶段产出

写一份安全检查清单，并在自己的项目中逐项落实。

## 11. 日志、观测与调试

### 学习目标

学会定位线上问题，而不是只会写业务逻辑。

### 必学内容

- 结构化日志：`log/slog`
- Request ID
- 状态码、耗时、方法、路径日志
- `net/http/pprof`
- 慢请求定位
- 错误链路追踪思路

### 推荐练习

- 为每个请求生成 Request ID
- 日志记录响应状态码和耗时
- 接入 `pprof`
- 压测接口并观察 CPU、内存

### 阶段产出

服务至少输出这些日志字段：

- `request_id`
- `method`
- `path`
- `status`
- `duration`
- `remote_addr`

## 12. 性能与并发

### 学习目标

理解 Go HTTP 服务的并发模型和性能瓶颈。

### 必学内容

- 每个请求由 goroutine 处理
- 共享数据的并发安全
- `sync.Mutex`
- `sync.RWMutex`
- `sync.Map`
- 连接复用
- Client 端连接池
- 压测工具：`wrk`、`ab`、`hey`

### 推荐练习

- 用 `hey` 压测 Todo API
- 故意制造 data race，再用 `go test -race` 发现问题
- 对比加锁前后的表现
- 调整 Client Transport 参数观察效果

### 阶段产出

能回答：

- 为什么内存 map 不能直接并发读写
- `http.Client` 是否应该每次请求都新建
- 服务端超时和客户端超时分别解决什么问题

## 13. 源码阅读路线

### 学习目标

通过源码理解 `net/http` 的设计思想，而不是死记 API。

### 推荐阅读顺序

1. `http.Handler`
2. `http.HandlerFunc`
3. `http.ServeMux`
4. `http.Server`
5. `serverHandler`
6. `conn`
7. `Request`
8. `Response`
9. `Client`
10. `Transport`

### 阅读重点

- 请求是如何被分发到 Handler 的
- 每个连接和每个请求之间是什么关系
- Server 如何处理超时
- Client 如何复用连接
- Transport 为什么很重要

### 阶段产出

画一张请求处理流程图：

```text
TCP connection
    -> http.Server
    -> ServeMux
    -> Middleware chain
    -> Handler
    -> ResponseWriter
    -> Client
```

## 14. 推荐项目路线

### 项目一：Todo API

重点训练：

- 路由
- JSON
- 错误处理
- Middleware
- 测试

### 项目二：短链接服务

重点训练：

- 重定向
- 状态码
- 数据存储
- 并发安全
- 访问统计

接口示例：

- `POST /links`
- `GET /{code}`
- `GET /links/{code}/stats`

### 项目三：文件上传服务

重点训练：

- Multipart
- 文件大小限制
- 静态文件
- 下载响应
- 安全校验

### 项目四：第三方 API 聚合服务

重点训练：

- HTTP Client
- Context 超时
- 并发请求
- 错误聚合
- 结果缓存

### 项目五：迷你 Web 框架

重点训练：

- 自己实现 Router
- 自己实现 Middleware 链
- Context 封装
- 错误处理
- 分组路由

注意：这个项目是为了理解框架原理，不建议在生产中替代成熟框架。

## 15. 推荐学习节奏

### 第 1 周：基础服务

- 学 HTTP 基础
- 学 `Handler`、`ServeMux`、`Request`、`ResponseWriter`
- 完成基础 Todo API

### 第 2 周：工程化

- 学 Middleware
- 学统一错误处理
- 学项目结构拆分
- 给 Todo API 加日志、鉴权、测试

### 第 3 周：生产级能力

- 学 `http.Server`
- 学超时配置
- 学优雅关闭
- 学 context
- 给服务补齐生产级启动和关闭逻辑

### 第 4 周：Client 与测试

- 学 `http.Client`
- 学 `httptest`
- 写第三方 API 客户端
- 完成 handler、middleware、client 测试

### 第 5 周：文件、安全、观测

- 学文件上传下载
- 学 Cookie、CORS、Token 鉴权
- 学 `slog` 和 `pprof`
- 做一次压测和性能分析

### 第 6 周：源码与综合项目

- 阅读 `net/http` 核心源码
- 完成短链接服务或 API 聚合服务
- 写项目 README
- 总结自己的 Web 服务模板

## 16. 必须养成的工程习惯

- 每个 handler 都要考虑错误路径
- 每个外部调用都要带 context
- 每个服务都要配置超时
- 每个响应都要有明确状态码
- JSON API 要有统一响应格式
- 不在 handler 中堆太多业务逻辑
- 不忽略 `resp.Body.Close()`
- 不每次请求都新建 `http.Client`
- 不把 panic 当成正常错误处理方式
- 测试不要只测成功路径

## 17. 面试准备清单

你应该能清楚回答这些问题：

- `http.Handler` 和 `http.HandlerFunc` 的关系是什么？
- `ResponseWriter.WriteHeader` 调用多次会怎样？
- Middleware 的本质是什么？
- 为什么生产环境不建议直接使用默认的 `http.ListenAndServe`？
- `ReadTimeout`、`WriteTimeout`、`IdleTimeout` 分别解决什么问题？
- `r.Context()` 什么时候会被取消？
- `http.Client` 为什么应该复用？
- `http.Transport` 的作用是什么？
- 如何测试一个 handler？
- 如何实现优雅关闭？
- 如何限制请求体大小？
- 如何处理文件上传？
- Go 的 HTTP 服务如何处理并发请求？

## 18. 最终目标

完成这条路线后，你应该能独立完成：

- 设计 RESTful JSON API
- 编写可测试的 handler 和 middleware
- 编写可靠的 HTTP client
- 配置生产级 HTTP server
- 实现优雅关闭和超时控制
- 处理文件上传下载
- 加入基础安全策略
- 使用日志、压测和 pprof 定位问题
- 阅读并理解 `net/http` 核心源码

## 19. 建议的最终作品

建议最终沉淀一个自己的 Go Web 后端模板：

```text
go-web-template/
├── cmd/
│   └── server/
├── internal/
│   ├── config/
│   ├── handler/
│   ├── middleware/
│   ├── model/
│   ├── service/
│   └── store/
├── pkg/
│   └── httpclient/
├── tests/
├── go.mod
└── README.md
```

模板应包含：

- `http.Server` 配置
- 优雅关闭
- 路由注册
- Middleware 链
- 统一错误响应
- JSON 工具函数
- 结构化日志
- 基础测试样例
- HTTP client 封装

做到这一步，你就不只是会用 `net/http`，而是已经具备 Go 后端工程师处理真实 Web 服务的基础能力。
