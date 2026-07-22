# 3. 使用标准库启动 HTTP 服务

本节目标：不使用 Gin，只用 Go 标准库启动一个 HTTP 服务。这样你会更清楚 Gin 到底解决了哪些重复工作。

Gin 是 Web 框架，但它不是凭空工作的。Gin 最终仍然运行在 Go 标准库 `net/http` 之上。

---

## 一、修改 main.go

把 `main.go` 改成：

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/ping", pingHandler)
    http.HandleFunc("/hello", helloHandler)

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

func helloHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    if name == "" {
        name = "guest"
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "message": "hello " + name,
    })
}
```

运行：

```bash
go run main.go
```

终端应该看到：

```text
server started at http://localhost:8080
```

---

## 二、访问接口

浏览器或接口工具访问：

```text
http://localhost:8080/ping
```

返回：

```json
{
  "message": "pong"
}
```

访问：

```text
http://localhost:8080/hello?name=alice
```

返回：

```json
{
  "message": "hello alice"
}
```

---

## 三、理解核心代码

### 1. 注册路由

```go
http.HandleFunc("/ping", pingHandler)
```

含义：

```text
当请求路径是 /ping 时，交给 pingHandler 函数处理。
```

### 2. 启动服务

```go
http.ListenAndServe(":8080", nil)
```

含义：

- `:8080`：监听本机 8080 端口。
- `nil`：使用标准库默认路由器。

### 3. handler 参数

```go
func pingHandler(w http.ResponseWriter, r *http.Request)
```

其中：

- `r`：客户端发来的请求。
- `w`：服务端写回给客户端的响应。

Gin 中的 `*gin.Context` 本质上也是围绕请求和响应做封装。

---

## 四、返回 JSON

设置响应头：

```go
w.Header().Set("Content-Type", "application/json")
```

写 JSON：

```go
json.NewEncoder(w).Encode(...)
```

如果不设置 `Content-Type`，客户端可能不知道响应体应该按 JSON 解析。

---

## 五、读取查询参数

请求：

```text
/hello?name=alice
```

读取：

```go
name := r.URL.Query().Get("name")
```

查询参数适合表达筛选、分页、搜索等条件。

后面在 Gin 中会写成：

```go
name := c.Query("name")
```

---

## 六、端口占用处理

如果运行时报错：

```text
bind: address already in use
```

说明 8080 端口已经被占用。

可以临时改成：

```go
http.ListenAndServe(":8081", nil)
```

或者关闭占用 8080 的程序。

---

## 七、停止服务

在终端中按：

```text
Ctrl + C
```

即可停止服务。

如果服务没有停止，再次运行可能会端口占用。

---

## 八、本节练习

增加一个接口：

```text
GET /time
```

返回当前时间：

```json
{
  "time": "2026-07-05T18:00:00+08:00"
}
```

提示：

```go
time.Now().Format(time.RFC3339)
```

---

## 九、本节验收

你应该能够回答：

- `http.HandleFunc` 做了什么？
- `http.ListenAndServe(":8080", nil)` 中的 `:8080` 是什么意思？
- `http.ResponseWriter` 用来做什么？
- `*http.Request` 里能拿到什么信息？
- Gin 为什么能比标准库写法更简洁？


