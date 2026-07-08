# 3. ServeMux 与路由匹配

本节目标：理解 `http.ServeMux` 的作用，并学会使用它组织多个路由。

---

## 一、ServeMux 是什么

`ServeMux` 是 Go 标准库提供的 HTTP 请求多路复用器。你可以先把它理解成“路由器”。

它做的事是：

```text
收到请求
-> 查看请求 Method 和 Path
-> 找到匹配的 Handler
-> 调用 Handler
```

如果没有 ServeMux，你只能把所有请求交给同一个 Handler，然后自己写大量 if/else 判断路径。

---

## 二、使用 NewServeMux

推荐写法：

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
		fmt.Fprintln(w, "hello")
	})

	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "ok")
	})

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

验证：

```bash
curl -i http://localhost:8080/hello
curl -i http://localhost:8080/health
```

---

## 三、为什么推荐自己创建 mux

你也可以使用：

```go
http.HandleFunc("/hello", handler)
http.ListenAndServe(":8080", nil)
```

这会使用全局默认路由器 `http.DefaultServeMux`。

学习示例可以用，但项目中更推荐：

```go
mux := http.NewServeMux()
```

原因是：

- 依赖更明确。
- 测试更方便。
- 不容易被其他包意外注册路由影响。
- 多个服务或子系统可以有自己的 mux。

---

## 四、Go 1.22 的路由模式

Go 1.22 开始，`ServeMux` 支持更清晰的模式：

```go
mux.HandleFunc("GET /todos", listTodos)
mux.HandleFunc("POST /todos", createTodo)
mux.HandleFunc("GET /todos/{id}", getTodo)
```

这比旧写法更接近真实 API 设计。

路径参数读取：

```go
id := r.PathValue("id")
```

完整示例：

```go
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	fmt.Fprintf(w, "user id: %s\n", id)
})
```

验证：

```bash
curl -i http://localhost:8080/users/42
```

响应：

```text
user id: 42
```

---

## 五、方法不匹配会怎样

如果注册：

```go
mux.HandleFunc("GET /hello", hello)
```

然后发送：

```bash
curl -i -X POST http://localhost:8080/hello
```

新版 `ServeMux` 会返回方法不允许，通常是 `405 Method Not Allowed`。

这比在 Handler 里手写：

```go
if r.Method != http.MethodGet {
	w.WriteHeader(http.StatusMethodNotAllowed)
	return
}
```

更简洁。

---

## 六、旧版本 Go 怎么办

如果你使用的 Go 版本低于 1.22，不支持 `"GET /path"` 这种模式。你可以写：

```go
mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	fmt.Fprintln(w, "hello")
})
```

路径参数也需要自己解析，或者使用第三方路由库。

本教程后续默认使用 Go 1.22 或更新版本。

---

## 七、路由设计建议

设计 API 时，尽量让路径表达资源：

```text
GET    /todos
POST   /todos
GET    /todos/{id}
PATCH  /todos/{id}
DELETE /todos/{id}
```

不要设计成：

```text
/getTodo
/createTodo
/deleteTodo
```

后者把动作塞进路径里，不利于形成统一风格。

---

## 八、本节检查点

请确认你能做到：

- 使用 `http.NewServeMux()` 创建路由器。
- 注册至少 3 个路由。
- 使用 `"GET /users/{id}"` 读取路径参数。
- 解释默认 mux 和自定义 mux 的区别。

