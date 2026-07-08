# 0. 路由组织与 Middleware 总览

本阶段目标：理解 `net/http` 中最重要的工程化能力之一：通过 `http.Handler` 组合中间件。

第 1 阶段你学了 Handler，第 2 阶段你写了 JSON API。现在会遇到新的问题：

- 每个请求都想记录日志，难道每个 Handler 都写一遍？
- 每个接口都要处理 panic，难道每个 Handler 都 defer recover？
- 很多接口都要鉴权，难道每个 Handler 都判断 Token？
- 每个响应都想加 Request ID，应该放在哪里？

这些横切逻辑就适合用 Middleware。

---

## 一、Middleware 是什么

Middleware 可以理解成包在 Handler 外面的一层逻辑。

请求进入时：

```text
client
-> logging middleware
-> auth middleware
-> recovery middleware
-> business handler
```

响应返回时，调用栈再一层层退回。

在 Go 中，典型中间件函数长这样：

```go
func middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// 请求进入 handler 前
		next.ServeHTTP(w, r)
		// handler 执行后
	})
}
```

---

## 二、本阶段文档安排

建议按下面顺序学习：

1. `1_Handler组合模型.md`
2. `2_日志耗时与RequestID中间件.md`
3. `3_Panic恢复与统一错误出口.md`
4. `4_简单鉴权与CORS.md`
5. `5_阶段练习_给Todo加入中间件.md`

---

## 三、中间件适合处理什么

适合放中间件：

- 请求日志。
- 请求耗时。
- Request ID。
- Panic Recovery。
- 鉴权。
- CORS。
- 请求体大小限制。
- 安全响应头。

不适合放中间件：

- 创建 Todo 的业务逻辑。
- 查询订单的数据库逻辑。
- 复杂业务校验。
- 与某一个具体接口强绑定的逻辑。

一句话：多个接口都需要、和 HTTP 流程相关的通用逻辑，更适合中间件。

---

## 四、本阶段达标标准

完成本阶段后，你应该能：

- 写出接收并返回 `http.Handler` 的中间件。
- 组合多个中间件。
- 实现日志中间件。
- 实现 panic recovery。
- 实现简单 Token 鉴权。
- 给 Todo API 加通用请求处理能力。

