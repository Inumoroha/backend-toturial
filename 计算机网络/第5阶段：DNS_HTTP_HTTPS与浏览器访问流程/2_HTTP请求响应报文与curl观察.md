# 2. HTTP 请求响应报文与 curl 观察

本节目标：理解 HTTP 请求和响应的基本格式，并使用 `curl -v` 观察真实请求。

---

## 一、HTTP 请求格式

HTTP 请求包括：

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

---

## 二、HTTP 响应格式

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

---

## 三、curl 观察请求

```bash
curl -v http://example.com
```

关注：

```text
> GET / HTTP/1.1
> Host: example.com
< HTTP/1.1 200 OK
< Content-Type: ...
```

`>` 表示客户端发出的内容，`<` 表示服务端返回的内容。

---

## 四、POST JSON 请求

```bash
curl -v -X POST http://127.0.0.1:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"alice"}'
```

参数说明：

- `-X POST`：指定方法。
- `-H`：添加 Header。
- `-d`：发送请求体。
- `-v`：显示详细过程。

---

## 五、Go HTTP Server 对应关系

```go
func handler(w http.ResponseWriter, r *http.Request) {
    method := r.Method
    path := r.URL.Path
    contentType := r.Header.Get("Content-Type")

    _ = method
    _ = path
    _ = contentType
}
```

Go 中：

- `r.Method` 对应请求方法。
- `r.URL.Path` 对应路径。
- `r.Header` 对应请求头。
- `r.Body` 对应请求体。
- `w.Header()` 设置响应头。
- `w.WriteHeader()` 设置状态码。

---

## 补充实验：写一个回显请求信息的 Go 服务

创建 `main.go`：

```go
package main

import (
    "encoding/json"
    "io"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/debug", func(w http.ResponseWriter, r *http.Request) {
        defer r.Body.Close()
        body, _ := io.ReadAll(io.LimitReader(r.Body, 1024*1024))

        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(map[string]any{
            "method":       r.Method,
            "path":         r.URL.Path,
            "query":        r.URL.RawQuery,
            "content_type": r.Header.Get("Content-Type"),
            "user_agent":   r.Header.Get("User-Agent"),
            "body":         string(body),
        })
    })

    log.Println("listen on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

启动：

```bash
go run .
```

发送请求：

```bash
curl -v "http://127.0.0.1:8080/debug?from=curl" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: req-001" \
  -d '{"name":"alice"}'
```

你应该观察三件事：

```text
curl 中以 > 开头的是客户端发出的请求。
curl 中以 < 开头的是服务端返回的响应。
Go 服务看到的 Method、Path、Header、Body 与 curl 发出的内容能对应上。
```

这个实验能把“HTTP 报文格式”和“Go handler 对象”连接起来。

---

## 补充排障：curl 输出卡在哪一段

`curl -v` 不只是用来看 Header，它能帮助你判断请求卡在哪一层。

常见片段：

```text
* Could not resolve host
```

表示 DNS 失败。

```text
* Trying 1.2.3.4:443...
```

如果长时间停在这里，通常是 TCP 建连慢或超时。

```text
* Connected to example.com
```

表示 TCP 已建立。

```text
* SSL connection using TLSv1.3
```

表示 TLS 已成功。

```text
< HTTP/1.1 502 Bad Gateway
```

表示已经进入 HTTP 层，且网关返回上游错误。

把这些信息分层看，排障会清楚很多。

---

## 六、常见问题

### 1. GET 能不能带 body？

协议层不是绝对禁止，但不推荐。很多代理、框架、缓存不会按 GET body 设计。

### 2. Header 大小无限吗？

不是。服务端和代理通常都有 Header 大小限制。

### 3. Content-Length 错了会怎样？

可能导致读取阻塞、请求失败或连接异常。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 写出 HTTP 请求和响应的基本结构。
- 使用 `curl -v` 观察请求和响应。
- 使用 curl 发送 JSON POST。
- 解释 Go 中 `http.Request` 和 `ResponseWriter` 对应 HTTP 报文的哪些部分。
