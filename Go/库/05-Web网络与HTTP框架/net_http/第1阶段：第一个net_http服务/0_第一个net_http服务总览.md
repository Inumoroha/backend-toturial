# 0. 第一个 net/http 服务总览

本阶段目标：用 Go 标准库写出第一个 HTTP 服务，并理解服务端最核心的几个概念：`Handler`、`HandlerFunc`、`ServeMux`、`Request`、`ResponseWriter`。

很多 Web 框架都把这些概念包装起来了，但底层仍然离不开它们。先学标准库，你以后再学 Gin、Echo、Fiber，会更容易理解框架在帮你做什么。

---

## 一、本阶段要解决什么问题

学完本阶段，你要能回答：

- Go 程序如何监听一个端口？
- 请求来了以后，谁负责找到对应处理函数？
- Handler 函数的两个参数分别是什么？
- 服务端如何读取请求信息？
- 服务端如何写响应？
- 为什么 `ListenAndServe` 一般写在 `main` 函数最后？

---

## 二、本阶段文档安排

建议按下面顺序学习：

1. `1_最小HTTP服务.md`
2. `2_Handler与HandlerFunc.md`
3. `3_ServeMux与路由匹配.md`
4. `4_Request与ResponseWriter入门.md`
5. `5_阶段练习_健康检查与问候服务.md`

---

## 三、本阶段核心模型

先记住这张图：

```text
client
  -> http.Server
  -> ServeMux
  -> Handler
  -> ResponseWriter
  -> client
```

用代码表达就是：

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /hello", helloHandler)
http.ListenAndServe(":8080", mux)
```

这里：

- `http.NewServeMux()` 创建路由分发器。
- `mux.HandleFunc(...)` 注册路由和处理函数。
- `http.ListenAndServe(":8080", mux)` 启动服务。

---

## 四、初学阶段不要急着引入框架

本阶段只用标准库。原因是：

- 标准库足够写出完整 HTTP 服务。
- 可以看清 Web 服务的基本组成。
- 能培养对请求和响应的直觉。
- 后面切换到框架时不会被魔法困住。

框架不是不能用，而是不要在还没理解 `net/http` 时就依赖框架。

---

## 五、本阶段达标标准

完成本阶段后，你应该能独立写出一个服务，包含：

- `/health` 返回 `ok`。
- `/hello` 返回固定文本。
- `/hello?name=alice` 返回带名字的文本。
- 不同路径返回不同内容。
- 用 curl 验证状态码和响应体。

