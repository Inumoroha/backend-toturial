# 1. Handler 组合模型

本节目标：真正理解中间件的本质：接收一个 Handler，返回一个新的 Handler。

---

## 一、从最小 Handler 开始

普通 Handler：

```go
func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello")
}
```

注册：

```go
mux.HandleFunc("GET /hello", hello)
```

如果要在执行 `hello` 前打印日志，最直接可以写：

```go
func hello(w http.ResponseWriter, r *http.Request) {
	log.Println(r.Method, r.URL.Path)
	fmt.Fprintln(w, "hello")
}
```

但如果有 50 个 Handler，就会重复 50 次。

---

## 二、把日志包在外面

中间件写法：

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println(r.Method, r.URL.Path)
		next.ServeHTTP(w, r)
	})
}
```

使用：

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /hello", hello)

handler := logging(mux)
http.ListenAndServe(":8080", handler)
```

请求先到 `logging` 返回的新 Handler，然后它再调用原来的 `mux`。

---

## 三、调用链是什么样

代码：

```go
handler := logging(mux)
```

请求进入时：

```text
logging handler
-> mux
-> matched route handler
```

如果再加一个 recovery：

```go
handler := recovery(logging(mux))
```

调用链：

```text
recovery
-> logging
-> mux
-> route handler
```

谁包在最外面，谁最先接到请求。

---

## 四、写一个 chain 函数

中间件多了以后，可以封装组合函数：

```go
type Middleware func(http.Handler) http.Handler

func chain(h http.Handler, middlewares ...Middleware) http.Handler {
	for i := len(middlewares) - 1; i >= 0; i-- {
		h = middlewares[i](h)
	}
	return h
}
```

使用：

```go
handler := chain(
	mux,
	recovery,
	logging,
)
```

这里调用顺序是：

```text
recovery -> logging -> mux
```

因为 `chain` 从后往前包。

---

## 五、为什么参数是 http.Handler

中间件不要只接收 `http.HandlerFunc`：

```go
func logging(next http.HandlerFunc) http.HandlerFunc
```

这样会限制它只能包装函数，不能包装 `ServeMux` 或自定义 Handler。

更通用的是：

```go
func logging(next http.Handler) http.Handler
```

因为：

- `ServeMux` 是 Handler。
- `HandlerFunc` 是 Handler。
- 自定义类型也可以是 Handler。

这就是接口的威力。

---

## 六、中间件中一定要调用 next 吗

不一定。

例如鉴权失败时，就不应该继续执行后面的 Handler：

```go
func auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer dev-token" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

如果不调用 `next.ServeHTTP`，请求就会在当前中间件结束。

---

## 七、本节检查点

请确认你能回答：

- 中间件为什么接收并返回 `http.Handler`？
- `next.ServeHTTP(w, r)` 的作用是什么？
- 多个中间件的执行顺序如何判断？
- 什么情况下中间件不应该调用 next？

