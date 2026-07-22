# 1. 安装 Gin 并启动第一个服务

本节目标：安装 Gin，启动第一个 Gin Web 服务，并访问 `GET /ping`。

---

## 一、确认当前项目

进入上一阶段创建的项目目录：

```bash
cd gin-lab
```

确认有 `go.mod`：

PowerShell：

```powershell
Get-ChildItem
```

Linux/macOS：

```bash
ls
```

应该能看到：

```text
go.mod
main.go
```

如果没有 `go.mod`，先执行：

```bash
go mod init gin-lab
```

---

## 二、安装 Gin

执行：

```bash
go get github.com/gin-gonic/gin
```

安装完成后，目录中会出现或更新：

```text
go.mod
go.sum
```

查看 `go.mod`，可以看到类似：

```go
require github.com/gin-gonic/gin v1.10.0
```

版本号可能不同，正常。

---

## 三、写第一个 Gin 服务

把 `main.go` 改成：

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })

    r.Run(":8080")
}
```

运行：

```bash
go run main.go
```

访问：

```http
GET http://localhost:8080/ping
```

返回：

```json
{
  "message": "pong"
}
```

---

## 四、逐行理解代码

### 1. 创建 Gin 引擎

```go
r := gin.Default()
```

`r` 可以理解为 Web 服务的路由引擎。后续所有接口都注册到它上面。

### 2. 注册 GET 路由

```go
r.GET("/ping", func(c *gin.Context) {
    ...
})
```

含义：

```text
当客户端用 GET 方法访问 /ping 时，执行后面的函数。
```

### 3. 返回 JSON

```go
c.JSON(http.StatusOK, gin.H{
    "message": "pong",
})
```

含义：

- HTTP 状态码是 `200`。
- 响应体是 JSON。
- `gin.H` 本质上是 `map[string]any` 的简写。

### 4. 启动服务

```go
r.Run(":8080")
```

含义：监听 8080 端口。

---

## 五、Gin 默认日志

启动服务后访问 `/ping`，终端会看到类似：

```text
[GIN] 2026/07/05 - 18:30:00 | 200 | 1.2ms | 127.0.0.1 | GET "/ping"
```

这行日志告诉你：

- 状态码：200。
- 耗时：1.2ms。
- 客户端 IP：127.0.0.1。
- 请求方法：GET。
- 请求路径：/ping。

这就是 `gin.Default()` 默认带来的请求日志能力。

---

## 六、常见问题

### 1. `go get` 下载失败

检查：

```bash
go env GOPROXY
```

如果网络慢，设置：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

### 2. 端口 8080 被占用

报错类似：

```text
bind: address already in use
```

可以临时改成：

```go
r.Run(":8081")
```

访问地址也要改成：

```text
http://localhost:8081/ping
```

### 3. 访问 404

检查：

- 是否访问了 `/ping`。
- 是否写成了 `/Ping`。
- 服务是否重新启动。
- 请求方法是否是 GET。

---

## 七、本节练习

增加一个接口：

```text
GET /hello
```

返回：

```json
{
  "message": "hello gin"
}
```

并在 `requests.http` 中增加请求：

```http
### hello
GET http://localhost:8080/hello
```

---

## 八、本节验收

你应该能够回答：

- `go get github.com/gin-gonic/gin` 做了什么？
- `gin.Default()` 返回的是什么？
- `r.GET` 的作用是什么？
- `c.JSON` 的两个参数分别是什么？
- Gin 默认日志能看出哪些信息？


