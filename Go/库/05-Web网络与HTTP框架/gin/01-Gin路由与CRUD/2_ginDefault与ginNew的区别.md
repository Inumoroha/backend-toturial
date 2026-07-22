# 2. gin.Default 与 gin.New 的区别

本节目标：理解 `gin.Default()` 和 `gin.New()` 的区别，知道什么时候使用默认中间件，什么时候自己组装中间件。

---

## 一、先看 `gin.Default()`

你在上一节写过：

```go
r := gin.Default()
```

它等价于创建一个 Gin 引擎，并默认挂载两个中间件：

```text
Logger
Recovery
```

可以简单理解为：

```text
gin.Default = gin.New + Logger + Recovery
```

---

## 二、Logger 中间件

Logger 用来打印请求日志。

例如访问：

```http
GET http://localhost:8080/ping
```

终端会输出：

```text
[GIN] 2026/07/05 - 18:30:00 | 200 | 1.2ms | 127.0.0.1 | GET "/ping"
```

学习阶段非常有用，因为你能确认：

- 请求有没有打到服务。
- 路径是否正确。
- 返回状态码是多少。
- 接口耗时大概多少。

---

## 三、Recovery 中间件

Recovery 用来捕获 panic，避免整个服务直接崩溃。

例如你写了：

```go
r.GET("/panic", func(c *gin.Context) {
    panic("something wrong")
})
```

如果使用 `gin.Default()`，请求 `/panic` 时 Gin 会返回 500，并在终端打印错误栈。

如果没有 Recovery，中小项目中一个未处理 panic 可能直接导致服务退出。

---

## 四、使用 `gin.New()`

如果你写：

```go
r := gin.New()
```

它会创建一个“干净”的 Gin 引擎，不自动挂载 Logger 和 Recovery。

如果你还想要它们，需要手动加：

```go
r := gin.New()
r.Use(gin.Logger())
r.Use(gin.Recovery())
```

这适合你需要完全控制中间件行为的场景，例如：

- 自定义日志格式。
- 使用 zap 替代 Gin 默认日志。
- 自定义 panic 响应格式。
- 测试时不希望终端输出太多日志。

---

## 五、学习阶段怎么选

建议：

```text
入门阶段：使用 gin.Default()
工程化阶段：逐渐改成 gin.New() + 自定义中间件
```

原因是：

- 入门阶段重点是理解路由和参数，不要过早陷入日志系统。
- 工程化阶段需要统一日志、错误响应和监控，这时再自定义更合适。

---

## 六、对比代码

默认写法：

```go
func main() {
    r := gin.Default()
    r.GET("/ping", ping)
    r.Run(":8080")
}
```

手动写法：

```go
func main() {
    r := gin.New()
    r.Use(gin.Logger())
    r.Use(gin.Recovery())

    r.GET("/ping", ping)
    r.Run(":8080")
}
```

这两段在基础效果上接近。

---

## 七、本节练习

完成：

1. 把 `gin.Default()` 改成 `gin.New()`。
2. 访问 `/ping`，观察终端是否还有请求日志。
3. 手动添加 `gin.Logger()` 和 `gin.Recovery()`。
4. 再次访问 `/ping`，观察日志是否恢复。
5. 增加 `/panic` 路由，观察 Recovery 的效果。

---

## 八、本节验收

你应该能够回答：

- `gin.Default()` 默认包含哪两个中间件？
- Logger 的作用是什么？
- Recovery 的作用是什么？
- `gin.New()` 适合什么场景？
- 为什么入门阶段推荐先用 `gin.Default()`？


