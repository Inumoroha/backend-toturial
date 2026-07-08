# 2. ServeMux 源码阅读路线

本节目标：理解 `ServeMux` 如何把请求分发给对应 Handler。

---

## 一、ServeMux 的角色

`ServeMux` 本身也是 Handler。

你可以这样启动：

```go
mux := http.NewServeMux()
http.ListenAndServe(":8080", mux)
```

因为它实现了：

```go
ServeHTTP(w http.ResponseWriter, r *http.Request)
```

---

## 二、核心流程

可以先用伪流程理解：

```text
ServeMux.ServeHTTP
-> 根据 Method、Host、Path 查找匹配规则
-> 找到 Handler
-> 设置路径参数
-> 调用 Handler.ServeHTTP
```

Go 1.22 之后 ServeMux 的路由能力更强，支持：

```go
"GET /todos/{id}"
```

---

## 三、阅读重点

阅读源码时重点看：

- 路由模式如何注册。
- 请求如何匹配模式。
- 方法不匹配时如何处理。
- 路径参数如何保存到 Request。
- 最终如何调用目标 Handler。

不要一上来陷入所有边界细节，先抓主流程。

---

## 四、为什么 ServeMux 也是 Handler 很重要

因为它可以被中间件包装：

```go
handler := logging(recovery(mux))
```

如果 ServeMux 不是 Handler，中间件组合会麻烦很多。

这体现了 `net/http` 设计的一致性：

```text
能处理请求的东西，统一抽象成 Handler。
```

---

## 五、本节检查点

请确认你能回答：

- ServeMux 为什么可以传给 `ListenAndServe`？
- ServeMux 的核心职责是什么？
- 路由匹配后最终调用了什么？
- 为什么中间件可以包装 ServeMux？

