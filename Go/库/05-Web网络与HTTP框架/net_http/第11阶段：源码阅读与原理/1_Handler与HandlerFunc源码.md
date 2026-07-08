# 1. Handler 与 HandlerFunc 源码

本节目标：从源码角度理解 `net/http` 最核心的抽象。

---

## 一、Handler 接口

标准库定义：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

这就是 Go HTTP 服务的核心。

只要一个类型有：

```go
ServeHTTP(http.ResponseWriter, *http.Request)
```

它就能处理 HTTP 请求。

---

## 二、HandlerFunc

源码思想：

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

这段代码的意义非常大：

```text
把普通函数适配成 Handler。
```

所以你才能写：

```go
mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "hello")
})
```

---

## 三、设计直觉

`Handler` 是接口，`HandlerFunc` 是适配器。

这种设计带来的好处：

- 函数可以当 Handler。
- 结构体可以当 Handler。
- 路由器可以当 Handler。
- 中间件可以包装 Handler。
- 测试可以直接调用 Handler。

---

## 四、阅读源码时看什么

不需要背源码。重点看：

- 接口定义有多小。
- 普通函数如何变成接口实现。
- `ServeHTTP` 是整个调用链的统一入口。

一旦你理解这个，Go Web 的很多库都会变得更透明。

---

## 五、本节检查点

请确认你能回答：

- `Handler` 为什么只需要一个方法？
- `HandlerFunc` 如何实现 `Handler`？
- 为什么这套设计让中间件变得自然？

