# 1. Context 接口与 emptyCtx

本节目标：理解 `Context` 接口和根 context 的实现。

接口：

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

所有具体 context 类型都实现这四个方法。

---

## 一、Background 和 TODO 返回什么

`context.Background()` 和 `context.TODO()` 返回的是根 context。

它们通常基于 `emptyCtx` 实现。

根 context 的特点：

```text
没有 deadline。
Done 为 nil。
Err 为 nil。
Value 返回 nil。
```

---

## 二、为什么 Done 可以是 nil

根 context 不会被取消。

所以它的 `Done()` 可以返回 nil。

在 select 中，nil channel 永远不会被选中：

```go
select {
case <-ctx.Done():
	return ctx.Err()
case <-time.After(time.Second):
	return nil
}
```

如果 ctx 是 `Background()`，`ctx.Done()` 不会触发。

---

## 三、Background 和 TODO 的行为差不多

它们的行为基本一致。

区别主要是语义：

```text
Background：明确的根。
TODO：暂时不知道该传什么。
```

源码中会用不同名字区分，方便调试和阅读。

---

## 四、本节达标标准

学完本节后，你应该能够做到：

- 背出 `Context` 接口四个方法。
- 知道根 context 不会取消。
- 知道 `Background` 和 `TODO` 的区别主要是语义。
- 理解 nil Done channel 在 select 中不会触发。

---

## 五、为什么接口只有四个方法

`context.Context` 的设计非常克制。

它没有提供：

- `Cancel()` 方法。
- `SetValue()` 方法。
- `SetDeadline()` 方法。
- `Children()` 方法。

这是故意的。

context 的使用者只能观察状态：

```text
有没有 deadline。
有没有取消。
取消原因是什么。
能不能读取某个请求级值。
```

取消动作由创建者持有的 `cancel` 函数控制，而不是任何拿到 ctx 的代码都能随便取消。

这能避免下游随意终止上游请求。

---

## 六、emptyCtx 的行为实验

可以写一个小程序观察根 context 的行为：

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func inspect(name string, ctx context.Context) {
	deadline, ok := ctx.Deadline()
	fmt.Println(name, "has deadline:", ok, deadline)
	fmt.Println(name, "err:", ctx.Err())
	fmt.Println(name, "value:", ctx.Value("missing"))

	select {
	case <-ctx.Done():
		fmt.Println(name, "done")
	case <-time.After(100 * time.Millisecond):
		fmt.Println(name, "not canceled")
	}
}

func main() {
	inspect("background", context.Background())
	inspect("todo", context.TODO())
}
```

运行：

```powershell
go run .
```

你会看到它们都不会取消，也没有 deadline。

---

## 七、为什么根 context 的 Done 是 nil 也没问题

在 Go 中，nil channel 的接收会永久阻塞。

所以在 `select` 里：

```go
select {
case <-ctx.Done():
	fmt.Println("done")
case <-time.After(time.Second):
	fmt.Println("timeout")
}
```

如果 `ctx.Done()` 是 nil，第一个分支永远不会准备好，`select` 会等待其他分支。

这正好符合根 context 的语义：

```text
它永远不会主动取消。
```

---

## 八、常见误解

### 1. Background 是全局变量吗？

你可以把它理解成根 context 的入口，但不要把它当成可以塞业务状态的全局容器。

### 2. TODO 会自动提醒我处理吗？

不会。`TODO()` 只是语义标记，不会在编译期或运行期报警。

所以团队里可以约定：提交前搜索 `context.TODO()`，确认每个 TODO 是否合理。

### 3. 根 context 能不能用于请求处理？

HTTP 请求里不应该用根 context 替代 `r.Context()`。否则客户端断开、请求超时等信号无法传给下游。

---

## 九、复习题

请用自己的话回答：

1. 为什么 `Context` 接口没有 `Cancel` 方法？
2. 为什么根 context 的 `Done()` 可以是 nil？
3. `Background()` 和 `TODO()` 行为相似，为什么还要分两个函数？
4. 在 HTTP handler 里直接使用 `context.Background()` 有什么问题？
