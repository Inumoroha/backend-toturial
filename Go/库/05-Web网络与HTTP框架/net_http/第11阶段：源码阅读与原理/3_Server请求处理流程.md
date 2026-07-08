# 3. Server 请求处理流程

本节目标：建立 `http.Server` 从接收连接到执行 Handler 的整体流程。

---

## 一、整体流程

简化流程：

```text
ListenAndServe
-> net.Listen
-> Accept connection
-> 为连接启动 goroutine
-> 读取 HTTP 请求
-> 找到 Handler
-> 调用 ServeHTTP
-> 写响应
```

真实源码更复杂，但主线就是这条。

---

## 二、每个请求都会有 goroutine 吗

Go HTTP Server 会为连接和请求处理创建 goroutine。你可以先建立直觉：

```text
多个请求可以并发处理。
Handler 中访问共享数据要考虑并发安全。
```

这就是为什么前面的内存 Todo map 要加锁。

---

## 三、serverHandler 的作用

源码中会有一个内部结构负责选择最终 Handler。

如果 Server 配置了 Handler：

```go
srv := &http.Server{Handler: mux}
```

就使用它。

如果 Handler 是 nil，就使用：

```go
http.DefaultServeMux
```

这解释了为什么：

```go
http.ListenAndServe(":8080", nil)
```

会使用默认路由器。

---

## 四、超时在哪里发挥作用

`http.Server` 的超时配置会影响连接读写过程：

- `ReadHeaderTimeout` 影响读 Header。
- `ReadTimeout` 影响读请求。
- `WriteTimeout` 影响写响应。
- `IdleTimeout` 影响空闲连接。

源码阅读时，可以关注这些字段在哪里被设置到连接 deadline 上。

---

## 五、本节检查点

请确认你能回答：

- `ListenAndServe` 大致做了哪些事？
- Handler 为 nil 时会使用什么？
- 为什么共享 map 要考虑并发安全？
- Server 超时和连接读写有什么关系？

