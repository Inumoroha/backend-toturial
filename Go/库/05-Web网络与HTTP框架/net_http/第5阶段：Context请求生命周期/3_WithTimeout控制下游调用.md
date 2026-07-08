# 3. WithTimeout 控制下游调用

本节目标：学会用 `context.WithTimeout` 给下游操作设置超时。

---

## 一、为什么下游调用要有超时

Handler 经常会调用：

- 数据库。
- Redis。
- 第三方 HTTP API。
- 内部微服务。
- 消息队列。

如果这些调用没有超时，接口可能一直卡住，连接和 goroutine 都被占用。

所以后端工程中有一条重要习惯：

```text
所有外部调用都要带 context，并设置合理超时。
```

---

## 二、创建带超时的 context

```go
ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
defer cancel()
```

注意父 context 是 `r.Context()`，不是 `context.Background()`。

这样它同时继承：

- 客户端断开取消。
- 当前操作 2 秒超时。

---

## 三、模拟下游调用

```go
func callSlowService(ctx context.Context) error {
	select {
	case <-time.After(3 * time.Second):
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

Handler：

```go
mux.HandleFunc("GET /call", func(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	if err := callSlowService(ctx); err != nil {
		http.Error(w, "downstream timeout", http.StatusGatewayTimeout)
		return
	}

	fmt.Fprintln(w, "ok")
})
```

这里下游需要 3 秒，但超时是 2 秒，所以会返回 `504 Gateway Timeout`。

---

## 四、为什么要 defer cancel

即使 context 最终会超时，也应该调用 cancel：

```go
defer cancel()
```

它可以及时释放与这个 context 相关的资源。

这是使用 `context.WithTimeout` 和 `context.WithCancel` 的标准习惯。

---

## 五、状态码怎么选

如果是客户端参数导致的错误：

```text
400 Bad Request
```

如果是服务端调用下游超时：

```text
504 Gateway Timeout
```

如果是服务端内部逻辑失败：

```text
500 Internal Server Error
```

下游超时不要简单都返回 500，`504` 更准确。

---

## 六、超时时间怎么定

没有一个通用数字永远正确。要结合：

- 接口整体超时目标。
- 下游服务平均耗时。
- 是否允许重试。
- 用户体验。
- 网关或负载均衡超时。

例如一个接口希望 1 秒内返回，数据库查询就不应该给 10 秒超时。

---

## 七、本节检查点

请确认你能回答：

- 为什么下游调用要设置超时？
- `context.WithTimeout(r.Context(), ...)` 和 `context.WithTimeout(context.Background(), ...)` 有什么区别？
- 为什么要 `defer cancel()`？
- 下游超时通常可以返回什么状态码？

