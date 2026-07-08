# 2. 类型安全的 Key 与封装函数

本节目标：学习安全使用 `context.WithValue` 的固定写法。

不要直接用字符串 key：

```go
ctx = context.WithValue(ctx, "request_id", requestID)
```

更推荐定义私有 key 类型。

---

## 一、为什么不要直接用 string

如果多个包都使用：

```go
"request_id"
```

就可能发生 key 冲突。

某个中间件设置的值，可能被另一个包覆盖。

使用自定义类型可以降低冲突风险。

---

## 二、推荐写法

```go
package requestctx

import "context"

type key string

const requestIDKey key = "request_id"

func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(requestIDKey).(string)
	return v, ok
}
```

这里 `key` 类型不导出，外部包无法随便构造同类型 key。

---

## 三、调用方式

设置：

```go
ctx = requestctx.WithRequestID(ctx, "req-001")
```

读取：

```go
requestID, ok := requestctx.RequestIDFromContext(ctx)
if ok {
	fmt.Println(requestID)
}
```

业务代码不需要知道底层 key 是什么。

---

## 四、封装的好处

封装函数有几个好处：

- 避免到处散落 key。
- 避免重复类型断言。
- 方便后续修改 value 类型。
- 让调用方更清楚语义。

不推荐在业务代码中到处写：

```go
ctx.Value(requestIDKey).(string)
```

因为类型断言失败会 panic。

推荐：

```go
requestID, ok := RequestIDFromContext(ctx)
```

---

## 五、多个值怎么组织

可以分别封装：

```go
WithRequestID
RequestIDFromContext

WithTraceID
TraceIDFromContext

WithAuthUser
AuthUserFromContext
```

不要把所有东西塞进一个大 map：

```go
ctx = context.WithValue(ctx, "all", map[string]any{...})
```

这样会让类型和边界更混乱。

---

## 六、本节练习

请创建一个 `requestctx` 包，完成：

1. `WithRequestID`
2. `RequestIDFromContext`
3. `WithTraceID`
4. `TraceIDFromContext`

然后在 main 中测试读写。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 使用私有 key 类型。
- 封装 `WithXxx` 和 `XxxFromContext`。
- 避免直接使用 string key。
- 避免业务代码到处类型断言。

---

## 八、推荐包结构

可以创建：

```text
internal/requestctx/context.go
```

专门放 context value 的封装。

这样 handler、service、logger 都可以复用：

```go
requestctx.WithRequestID(ctx, id)
requestctx.RequestIDFromContext(ctx)
```

不要把 key 分散在多个业务文件里。

---

## 九、完整示例

```go
package requestctx

import "context"

type key struct {
	name string
}

var requestIDKey = key{name: "request_id"}

func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	value, ok := ctx.Value(requestIDKey).(string)
	return value, ok
}
```

使用结构体 key 也可以，关键是不要导出 key。

---

## 十、读取时提供默认值

有时日志里希望没有 request id 时返回默认值：

```go
func RequestIDOrDefault(ctx context.Context) string {
	requestID, ok := RequestIDFromContext(ctx)
	if !ok || requestID == "" {
		return "unknown"
	}
	return requestID
}
```

这样日志函数更简单。

---

## 十一、常见错误

### 1. key 导出

如果外部包能直接使用你的 key，就可能绕过封装。

### 2. 读取函数 panic

不推荐：

```go
return ctx.Value(key).(string)
```

### 3. key 复用不同类型

同一个 key 一会儿存 string，一会儿存 int，会让读取方混乱。

---

## 十二、本节练习

创建 `requestctx` 包，实现：

```go
WithRequestID
RequestIDFromContext
RequestIDOrDefault
WithTraceID
TraceIDFromContext
TraceIDOrDefault
```

然后写一个小 main 验证默认值和正常值。
