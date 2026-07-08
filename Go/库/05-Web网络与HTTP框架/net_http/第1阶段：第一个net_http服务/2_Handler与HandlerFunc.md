# 2. Handler 与 HandlerFunc

本节目标：理解 `net/http` 的核心抽象：`http.Handler`。只要理解了 Handler，后面的路由、中间件、测试都会变得清楚。

---

## 一、Handler 是什么

标准库中的定义是：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

这句话非常重要。

它的意思是：只要一个类型实现了 `ServeHTTP(http.ResponseWriter, *http.Request)` 方法，它就是一个 HTTP 处理器。

请求来了以后，Go 最终会调用某个 Handler 的 `ServeHTTP` 方法。

---

## 二、自己实现一个 Handler

先不用 `HandleFunc`，我们手动写一个类型：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

type helloHandler struct{}

func (h helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello from handler")
}

func main() {
	var h helloHandler

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", h))
}
```

访问任意路径：

```bash
curl -i http://localhost:8080/hello
curl -i http://localhost:8080/anything
```

都会返回：

```text
hello from handler
```

因为我们把整个服务的 Handler 都设置成了 `helloHandler`。

---

## 三、HandlerFunc 是什么

你之前写过：

```go
http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello")
})
```

这里的函数为什么也能当 Handler 用？

因为标准库定义了：

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

也就是说，`HandlerFunc` 是一个函数类型，并且它实现了 `ServeHTTP` 方法。

这就是 Go 里非常漂亮的适配方式：

```text
普通函数
-> 转成 HandlerFunc
-> 拥有 ServeHTTP 方法
-> 满足 Handler 接口
```

---

## 四、把函数显式转成 HandlerFunc

示例：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello")
}

func main() {
	h := http.HandlerFunc(hello)

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", h))
}
```

这里 `hello` 本来只是一个普通函数。经过 `http.HandlerFunc(hello)` 转换后，它就满足了 `http.Handler` 接口。

---

## 五、为什么这个抽象重要

因为 `net/http` 的很多能力都是围绕 Handler 组合出来的。

路由器是 Handler：

```go
mux := http.NewServeMux()
```

中间件接收 Handler，返回 Handler：

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println(r.Method, r.URL.Path)
		next.ServeHTTP(w, r)
	})
}
```

测试 Handler：

```go
handler.ServeHTTP(recorder, request)
```

所以你可以把 `net/http` 的核心理解成一句话：

```text
HTTP 服务就是一层层 Handler 的组合。
```

---

## 六、Handler 和业务函数的区别

Handler 面对的是 HTTP 世界：

- 状态码。
- Header。
- Request。
- ResponseWriter。
- Cookie。
- Body。

业务函数面对的是业务世界：

- 创建 Todo。
- 查询用户。
- 删除订单。
- 生成短链接。

初学时可以把业务写在 Handler 里。但项目变大后，建议拆分：

```text
Handler 负责 HTTP 输入输出。
Service 负责业务规则。
Repository 或 Store 负责数据访问。
```

这样测试和维护会更容易。

---

## 七、本节检查点

请确认你能回答：

- `http.Handler` 接口长什么样？
- 一个类型怎样才算 Handler？
- `http.HandlerFunc` 为什么能把函数当成 Handler？
- 为什么中间件通常接收并返回 `http.Handler`？

