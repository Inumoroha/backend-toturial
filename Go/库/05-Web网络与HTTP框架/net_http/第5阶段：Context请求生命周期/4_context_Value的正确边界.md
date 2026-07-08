# 4. context.Value 的正确边界

本节目标：理解 `context.Value` 适合放什么，不适合放什么。

---

## 一、context.Value 能做什么

`context.Context` 除了取消和超时，还能携带键值：

```go
ctx := context.WithValue(r.Context(), requestIDKey, "req-123")
```

读取：

```go
id, _ := ctx.Value(requestIDKey).(string)
```

这在 Request ID、Trace ID、当前用户 ID 等场景中很常见。

---

## 二、适合放进 context 的内容

适合：

- Request ID。
- Trace ID。
- 当前认证用户 ID。
- 租户 ID。
- 请求范围内的轻量元信息。

它们的特点是：

- 跟一次请求强相关。
- 通常跨多个层需要访问。
- 不是主要业务输入。
- 数据量小。

---

## 三、不适合放进 context 的内容

不适合：

- 完整请求 Body。
- 大型业务对象。
- 数据库连接池。
- 配置对象。
- 可选参数。
- 函数本来应该显式传递的业务参数。

不推荐：

```go
ctx = context.WithValue(ctx, "todo_title", title)
service.CreateTodo(ctx)
```

推荐：

```go
service.CreateTodo(ctx, title)
```

业务参数应该显式传递，这样函数签名更清楚。

---

## 四、不要用 string 作为 key

不推荐：

```go
context.WithValue(ctx, "request_id", id)
```

不同包可能使用同样字符串，造成冲突。

推荐定义私有类型：

```go
type contextKey string

const requestIDKey contextKey = "request_id"
```

这样可以降低 key 冲突概率。

---

## 五、封装读取函数

推荐：

```go
func requestIDFromContext(ctx context.Context) string {
	id, _ := ctx.Value(requestIDKey).(string)
	return id
}
```

这样其他代码不需要知道 key 的细节。

---

## 六、本节检查点

请确认你能回答：

- `context.Value` 适合放哪些数据？
- 为什么业务参数不应该塞进 context？
- 为什么 key 不推荐直接用 string？
- 为什么要封装读取函数？

