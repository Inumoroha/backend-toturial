# 3. Done、Err、Deadline、Value

本节目标：通过代码理解 `Context` 四个方法的日常用法。

前面看过接口定义：

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

这一节把它们放进真实代码里。

---

## 一、Done：监听取消信号

最常见写法：

```go
func work(ctx context.Context) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
			fmt.Println("working")
			time.Sleep(300 * time.Millisecond)
		}
	}
}
```

当 context 被取消时，`ctx.Done()` 会被关闭，`select` 中对应分支就会执行。

---

## 二、Err：判断为什么结束

`ctx.Err()` 常见结果：

```go
context.Canceled
context.DeadlineExceeded
```

示例：

```go
err := work(ctx)
if errors.Is(err, context.Canceled) {
	fmt.Println("task canceled")
}
if errors.Is(err, context.DeadlineExceeded) {
	fmt.Println("task timeout")
}
```

建议使用 `errors.Is` 判断，而不是直接比较字符串。

---

## 三、Deadline：读取截止时间

某些函数想知道剩余时间，可以读 deadline：

```go
func printRemaining(ctx context.Context) {
	deadline, ok := ctx.Deadline()
	if !ok {
		fmt.Println("no deadline")
		return
	}

	fmt.Println("remaining:", time.Until(deadline))
}
```

注意：`Deadline()` 只是读取，不会设置 deadline。

设置 deadline 要用：

```go
context.WithDeadline(...)
context.WithTimeout(...)
```

---

## 四、Value：读取请求级数据

先看简单写法：

```go
type contextKey string

const requestIDKey contextKey = "request_id"

ctx := context.WithValue(context.Background(), requestIDKey, "req-001")
fmt.Println(ctx.Value(requestIDKey))
```

更推荐封装：

```go
func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(requestIDKey).(string)
	return v, ok
}
```

这样业务代码不需要到处直接写 key。

---

## 五、完整示例

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func work(ctx context.Context) error {
	deadline, ok := ctx.Deadline()
	if ok {
		fmt.Println("deadline:", deadline.Format(time.RFC3339Nano))
	}

	select {
	case <-time.After(2 * time.Second):
		fmt.Println("work done")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	err := work(ctx)
	if errors.Is(err, context.DeadlineExceeded) {
		fmt.Println("timeout")
		return
	}
	if err != nil {
		fmt.Println("error:", err)
		return
	}
}
```

任务需要 2 秒，但 context 只有 1 秒，所以会输出超时。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 用 `ctx.Done()` 等待取消。
- 用 `ctx.Err()` 返回取消原因。
- 用 `errors.Is` 判断 `context.Canceled` 和 `context.DeadlineExceeded`。
- 用 `Deadline()` 查看是否有截止时间。
- 用封装函数安全读写 request id。

---

## 七、Done 和 Err 的配套使用

通常不要只写：

```go
<-ctx.Done()
```

还要读取：

```go
return ctx.Err()
```

因为调用方需要知道：

```text
是主动取消？
还是超时？
```

这两个原因会影响日志级别和 HTTP 响应。

---

## 八、Deadline 的实际用途

`Deadline()` 常用于：

- 打日志。
- 判断剩余时间是否足够。
- 给下游协议传递截止时间。

示例：

```go
func hasEnoughTime(ctx context.Context, need time.Duration) bool {
	deadline, ok := ctx.Deadline()
	if !ok {
		return true
	}
	return time.Until(deadline) > need
}
```

如果请求只剩 10ms，就没必要发起预计 500ms 的下游调用。

---

## 九、Value 的最小安全模式

```go
type contextKey string

const requestIDKey contextKey = "request_id"

func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) string {
	value, ok := ctx.Value(requestIDKey).(string)
	if !ok {
		return "unknown"
	}
	return value
}
```

学习阶段先这样写，后续再根据项目规范调整。

---

## 十、常见错误

### 1. 从 Done 里读值

`Done()` channel 传递的是关闭信号，不是错误值。

### 2. 不返回 ctx.Err

调用方无法判断取消原因。

### 3. Deadline 当作设置函数

它只是读取。

### 4. Value 到处强制类型断言

容易 panic。

---

## 十一、本节练习

写一个 `run(ctx)`：

1. 开始时打印剩余 deadline。
2. 每 200ms 打印一次 running。
3. ctx 取消后返回 `ctx.Err()`。
4. main 中用 1 秒 timeout 调用。
5. 打印最终错误类型。
