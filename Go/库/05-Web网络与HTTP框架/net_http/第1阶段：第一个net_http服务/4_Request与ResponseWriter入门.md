# 4. Request 与 ResponseWriter 入门

本节目标：理解 Handler 函数里的两个参数：`http.ResponseWriter` 和 `*http.Request`。

几乎每个 Handler 都长这样：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	// read request from r
	// write response to w
}
```

可以先记一句话：

```text
r 表示客户端发来的请求。
w 用来写回服务端响应。
```

---

## 一、读取请求方法和路径

示例：

```go
mux.HandleFunc("GET /debug", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "method=%s\n", r.Method)
	fmt.Fprintf(w, "path=%s\n", r.URL.Path)
})
```

访问：

```bash
curl -i http://localhost:8080/debug
```

响应：

```text
method=GET
path=/debug
```

---

## 二、读取 Query 参数

示例：

```go
mux.HandleFunc("GET /search", func(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query().Get("q")
	if q == "" {
		http.Error(w, "missing q", http.StatusBadRequest)
		return
	}

	fmt.Fprintf(w, "search: %s\n", q)
})
```

访问：

```bash
curl -i "http://localhost:8080/search?q=go"
```

如果不传 `q`：

```bash
curl -i http://localhost:8080/search
```

会返回 `400 Bad Request`。

---

## 三、写响应体

最直接的方式：

```go
w.Write([]byte("hello\n"))
```

更常用的是：

```go
fmt.Fprintln(w, "hello")
```

因为 `ResponseWriter` 实现了 `io.Writer`，所以很多写入函数都能直接使用它。

---

## 四、设置 Header

响应 JSON 时要设置：

```go
w.Header().Set("Content-Type", "application/json")
```

注意：Header 要在写状态码或写响应体之前设置。

正确顺序：

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
w.Write([]byte(`{"id":1}`))
```

错误顺序：

```go
w.WriteHeader(http.StatusCreated)
w.Header().Set("Content-Type", "application/json")
```

后设置的 Header 可能不会生效，因为响应头已经发送出去了。

---

## 五、设置状态码

默认情况下，如果你直接写响应体：

```go
fmt.Fprintln(w, "ok")
```

Go 会自动返回 `200 OK`。

如果要返回其他状态码，需要在写 Body 前调用：

```go
w.WriteHeader(http.StatusCreated)
```

示例：

```go
mux.HandleFunc("POST /todos", func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusCreated)
	fmt.Fprintln(w, "created")
})
```

验证：

```bash
curl -i -X POST http://localhost:8080/todos
```

---

## 六、WriteHeader 只能有效调用一次

一旦状态码写出，后面再调用 `WriteHeader` 不会改变已经发送的状态码。

例如：

```go
w.WriteHeader(http.StatusCreated)
w.WriteHeader(http.StatusInternalServerError)
fmt.Fprintln(w, "body")
```

客户端看到的仍然是第一次写出的状态码。

所以 Handler 中要先判断错误，再写成功响应：

```go
if err != nil {
	http.Error(w, "something wrong", http.StatusInternalServerError)
	return
}

w.WriteHeader(http.StatusCreated)
fmt.Fprintln(w, "created")
```

---

## 七、http.Error 入门

标准库提供了一个简单错误响应函数：

```go
http.Error(w, "missing q", http.StatusBadRequest)
```

它会设置状态码，并写入文本错误。

前期可以使用它。后面写 JSON API 时，我们会自己封装统一 JSON 错误响应。

---

## 八、本节完整示例

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /search", func(w http.ResponseWriter, r *http.Request) {
		q := r.URL.Query().Get("q")
		if q == "" {
			http.Error(w, "missing q", http.StatusBadRequest)
			return
		}

		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		fmt.Fprintf(w, "search: %s\n", q)
	})

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 九、本节检查点

请确认你能回答：

- `*http.Request` 里常读哪些信息？
- `ResponseWriter` 用来做什么？
- Header 应该什么时候设置？
- `WriteHeader` 调多次会怎样？
- 直接写 Body 时默认状态码是什么？

