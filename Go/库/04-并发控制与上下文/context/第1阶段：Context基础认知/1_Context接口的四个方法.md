# 1. Context 接口的四个方法

本节目标：理解 `context.Context` 的接口定义，以及每个方法的作用。

先看接口：

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

只有四个方法，但它们覆盖了 context 的核心能力。

---

## 一、Deadline

`Deadline` 用来获取截止时间。

```go
deadline, ok := ctx.Deadline()
```

如果 `ok == false`，说明这个 context 没有截止时间。

示例：

```go
func printDeadline(ctx context.Context) {
	deadline, ok := ctx.Deadline()
	if !ok {
		fmt.Println("no deadline")
		return
	}

	fmt.Println("deadline:", deadline)
}
```

`Deadline` 不会阻塞，它只是告诉你有没有截止时间。

---

## 二、Done

`Done` 返回一个只读 channel：

```go
Done() <-chan struct{}
```

当 context 被取消或超时时，这个 channel 会被关闭。

典型用法：

```go
select {
case <-ctx.Done():
	return ctx.Err()
case result := <-resultCh:
	return result
}
```

注意：不是从 `Done()` 里读取具体值，而是等待它关闭。

---

## 三、Err

`Err` 用来查看 context 为什么结束。

常见返回值：

```go
context.Canceled
context.DeadlineExceeded
```

示例：

```go
if err := ctx.Err(); err != nil {
	fmt.Println("context finished:", err)
}
```

如果 context 还没有被取消或超时，`Err()` 返回 `nil`。

---

## 四、Value

`Value` 用来读取请求级元数据：

```go
requestID := ctx.Value(requestIDKey)
```

它不适合存业务参数。

适合放：

- request id。
- trace id。
- 用户认证信息。
- 日志字段。

不适合放：

- userID 这种业务必需参数。
- 数据库连接。
- service 对象。
- 配置项。

---

## 五、为什么 Context 是接口

`context.Context` 是接口，不是具体结构体。

因为不同 context 需要不同能力：

- `emptyCtx`：根 context，没有取消、没有值、没有 deadline。
- `cancelCtx`：支持取消。
- `timerCtx`：支持超时和 deadline。
- `valueCtx`：支持存取值。

它们都实现同一个接口，所以函数只需要接收：

```go
ctx context.Context
```

而不需要关心具体类型。

---

## 六、练习

写一个函数：

```go
func inspectContext(ctx context.Context) {
	deadline, ok := ctx.Deadline()
	fmt.Println("has deadline:", ok)
	if ok {
		fmt.Println("deadline:", deadline)
	}

	fmt.Println("err:", ctx.Err())
}
```

分别传入：

```go
context.Background()
context.TODO()
```

观察输出。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 背出 `Context` 接口的四个方法。
- 知道 `Done()` 返回的是只读 channel。
- 知道 `Err()` 用来判断取消原因。
- 知道 `Deadline()` 不会设置超时，只是读取超时。
- 知道 `Value()` 不能滥用。

---

## 八、四个方法的使用频率

实际项目中使用频率大致是：

```text
Done：非常常用。
Err：非常常用。
Deadline：偶尔使用。
Value：谨慎使用。
```

你最应该熟练的是：

```go
select {
case <-ctx.Done():
	return ctx.Err()
}
```

`Deadline()` 多用于判断剩余时间或调试。

`Value()` 只用于请求级元数据。

---

## 九、完整 inspect 示例

```go
func inspect(ctx context.Context) {
	if deadline, ok := ctx.Deadline(); ok {
		fmt.Println("deadline:", deadline)
	} else {
		fmt.Println("no deadline")
	}

	select {
	case <-ctx.Done():
		fmt.Println("done:", ctx.Err())
	default:
		fmt.Println("not done")
	}
}
```

这里用 `default` 是为了非阻塞检查。

如果没有 default，`Background()` 的 `Done()` 可能永远不返回。

---

## 十、常见误解

### 1. Deadline 会设置超时？

不会。它只是读取 deadline。

设置超时要用 `WithTimeout` 或 `WithDeadline`。

### 2. Done 返回错误？

不会。Done 返回 channel。

错误要通过 `Err()` 读取。

### 3. Value 是泛型 map？

不是。它只是提供请求级值查找能力。

---

## 十一、本节练习

写一个 `debugContext(ctx)`：

1. 打印是否有 deadline。
2. 打印是否已经取消。
3. 如果已经取消，打印 `Err()`。
4. 读取 request id，如果没有就打印 unknown。

分别传入普通 ctx、timeout ctx、带 request id 的 ctx。
