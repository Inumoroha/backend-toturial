# 02-HTTP 协议与 Go 服务

## 学习目标

掌握 HTTP 请求、响应、状态码、Header、Keep-Alive，并能用 Go 写出基础 HTTP 服务。

## 一、HTTP 请求结构

一个 HTTP 请求包括：

```text
请求行
请求头
空行
请求体
```

示例：

```http
GET /users/1 HTTP/1.1
Host: api.example.com
User-Agent: curl/8.0
Accept: */*
```

POST 示例：

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 18

{"name":"alice"}
```

## 二、HTTP 响应结构

响应包括：

```text
状态行
响应头
空行
响应体
```

示例：

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 24

{"id":1,"name":"alice"}
```

## 三、常见状态码

| 状态码 | 含义 | 后端场景 |
| --- | --- | --- |
| 200 | 成功 | 请求正常处理 |
| 201 | 已创建 | 创建资源成功 |
| 204 | 成功但无响应体 | 删除成功 |
| 301/302 | 重定向 | HTTP 跳 HTTPS |
| 400 | 请求格式错误 | 参数无法解析 |
| 401 | 未认证 | 未登录 |
| 403 | 无权限 | 登录但无权限 |
| 404 | 不存在 | 路由或资源不存在 |
| 409 | 冲突 | 重复创建、版本冲突 |
| 429 | 请求过多 | 限流 |
| 500 | 服务内部错误 | 程序异常 |
| 502 | 网关无法连接后端 | Nginx 到上游失败 |
| 503 | 服务不可用 | 过载、维护、无健康实例 |
| 504 | 网关等待后端超时 | 上游响应太慢 |

## 四、Header

Header 用来传递元信息。

常见 Header：

- `Content-Type`
- `Content-Length`
- `Authorization`
- `Cookie`
- `Set-Cookie`
- `User-Agent`
- `X-Request-Id`
- `X-Forwarded-For`

后端服务中，`X-Request-Id` 对链路追踪和日志关联非常有用。

## 五、HTTP Keep-Alive

HTTP/1.1 默认支持长连接。多个请求可以复用同一个 TCP 连接。

好处：

- 减少 TCP 握手开销。
- 减少 TLS 握手开销。
- 提升吞吐。

注意：

- Keep-Alive 是 HTTP 连接复用。
- TCP Keepalive 是 TCP 层探测空闲连接是否存活。
- 二者不是一回事。

## 六、Go HTTP Server 示例

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusNoContent)
    })

    mux.HandleFunc("/users/1", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(User{ID: 1, Name: "alice"})
    })

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 2 * time.Second,
        ReadTimeout:       5 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Println("listen on :8080")
    log.Fatal(srv.ListenAndServe())
}
```

## 七、实验：curl 观察 HTTP

```bash
curl -v http://127.0.0.1:8080/users/1
```

观察：

- 请求方法和路径。
- 请求 Header。
- 响应状态码。
- 响应 Header。
- 响应体。

## 八、Go 后端注意点

服务端：

- 不要直接使用默认 `http.ListenAndServe` 处理生产服务，建议显式配置 `http.Server`。
- 设置 `ReadHeaderTimeout` 防止慢速 Header 攻击。
- 设置 `ReadTimeout` 和 `WriteTimeout` 防止连接长期占用。
- 记录请求方法、路径、状态码、耗时、错误。

客户端：

- 复用 `http.Client`。
- 设置超时。
- 关闭 `resp.Body`。

## 九、练习题

1. HTTP 请求由哪几部分组成？
2. 502、503、504 分别常见于什么场景？
3. HTTP Keep-Alive 和 TCP Keepalive 有什么区别？
4. Go HTTP Server 为什么要配置超时？

## 十、验收标准

你能写出带超时配置的 Go HTTP Server，并用 `curl -v` 解释一次请求和响应。

