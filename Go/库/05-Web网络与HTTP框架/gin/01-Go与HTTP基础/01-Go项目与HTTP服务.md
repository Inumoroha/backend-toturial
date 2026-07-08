# 01-01 Go 项目与 HTTP 服务

本阶段目标：用 Go 标准库写一个最小 HTTP 服务，理解 Web 请求从浏览器或客户端进入后端之后发生了什么。Gin 的底层仍然基于 Go 的 `net/http`，所以这一阶段非常重要。

## 一、创建项目

在你的学习目录中创建项目：

```bash
mkdir gin-learning
cd gin-learning
go mod init gin-learning
```

创建文件：

```text
main.go
```

写入最小程序：

```go
package main

import "fmt"

func main() {
    fmt.Println("hello go backend")
}
```

运行：

```bash
go run main.go
```

看到输出后，说明 Go 项目基本环境正常。

## 二、理解 HTTP 请求和响应

一次 HTTP 请求通常包含：

- 请求方法：`GET`、`POST`、`PUT`、`DELETE`
- 请求路径：例如 `/users`
- 请求头：例如 `Content-Type: application/json`
- 请求体：例如 JSON 数据

一次 HTTP 响应通常包含：

- 状态码：例如 `200`、`400`、`404`、`500`
- 响应头：例如 `Content-Type`
- 响应体：通常是 JSON

后端开发的本质，就是接收请求、处理业务、返回响应。

## 三、使用标准库创建 HTTP 服务

把 `main.go` 改成：

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

func main() {
    http.HandleFunc("/ping", pingHandler)
    http.HandleFunc("/users", usersHandler)

    log.Println("server started at http://localhost:8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatal(err)
    }
}

func pingHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "message": "pong",
    })
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Username: "alice", Email: "alice@example.com"},
        {ID: 2, Username: "bob", Email: "bob@example.com"},
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

运行：

```bash
go run main.go
```

访问：

```text
GET http://localhost:8080/ping
GET http://localhost:8080/users
```

## 四、理解 handler

标准库中的 handler 函数形态是：

```go
func(w http.ResponseWriter, r *http.Request)
```

其中：

- `r *http.Request` 表示客户端发来的请求。
- `w http.ResponseWriter` 用来写回响应。

在 Gin 中，你会看到类似的概念，只是 Gin 把很多操作封装到了 `*gin.Context` 里面。

## 五、处理不同请求方法

继续改造 `usersHandler`：

```go
func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        users := []User{
            {ID: 1, Username: "alice", Email: "alice@example.com"},
            {ID: 2, Username: "bob", Email: "bob@example.com"},
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(users)
    case http.MethodPost:
        var req User
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "invalid json", http.StatusBadRequest)
            return
        }

        req.ID = 3
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(req)
    default:
        http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
    }
}
```

测试创建用户：

```http
POST http://localhost:8080/users
Content-Type: application/json

{
  "username": "cindy",
  "email": "cindy@example.com"
}
```

你会发现标准库能写 Web 服务，但代码逐渐变得繁琐。Gin 的价值就是让路由、参数绑定、JSON 响应、中间件等操作更清晰。

## 六、常见问题

### 1. 端口被占用

如果看到 `bind: address already in use`，说明 8080 端口已经被其他程序占用。可以换成：

```go
http.ListenAndServe(":8081", nil)
```

### 2. JSON 字段名不对

如果结构体没有写 `json` tag，返回字段可能是大写的：

```go
Username string
```

返回可能是：

```json
{
  "Username": "alice"
}
```

加上 tag 后：

```go
Username string `json:"username"`
```

返回就是：

```json
{
  "username": "alice"
}
```

## 七、阶段练习

请基于标准库完成下面接口：

- `GET /ping` 返回 `{"message":"pong"}`
- `GET /users` 返回用户列表
- `POST /users` 接收 JSON 并返回创建后的用户
- 当请求方法不支持时返回 `405`
- 当 JSON 解析失败时返回 `400`

## 八、验收清单

你应该能够回答：

- `http.Request` 里有什么？
- `http.ResponseWriter` 用来做什么？
- `Content-Type: application/json` 为什么重要？
- `GET` 和 `POST` 在语义上有什么区别？
- Gin 相比标准库主要简化了哪些事情？

