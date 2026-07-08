# 3. timerCtx 超时实现

本节目标：理解 `WithTimeout` 和 `WithDeadline` 背后的 `timerCtx`。

`timerCtx` 可以理解成：

```text
在 cancelCtx 的基础上，多了 deadline 和 timer。
```

---

## 一、WithTimeout 的本质

`WithTimeout(parent, d)` 本质上等价于：

```go
WithDeadline(parent, time.Now().Add(d))
```

所以源码重点在 `WithDeadline`。

---

## 二、timerCtx 做了什么

创建 deadline context 时，大致会：

```text
计算截止时间。
创建 timer。
timer 到期后调用 cancel。
如果手动 cancel，停止 timer。
```

这就是为什么超时时 `ctx.Done()` 会自动关闭。

timer 到期后，内部调用取消逻辑，错误原因是：

```go
context.DeadlineExceeded
```

---

## 三、为什么 defer cancel 仍然重要

如果你写：

```go
ctx, cancel := context.WithTimeout(parent, 10*time.Second)
defer cancel()
```

任务 100ms 就完成了。

如果不调用 cancel，timer 可能要等到 10 秒后才释放。

调用 cancel 可以提前停止 timer，释放相关资源。

---

## 四、父 deadline 更早怎么办

如果父 context 1 秒后超时，而你给子 context 设置 10 秒 timeout。

子 context 不会真的活 10 秒。

父 context 取消会传播给子 context。

这也是为什么超时预算要从入口开始设计。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 解释 `WithTimeout` 和 `WithDeadline` 的关系。
- 知道 `timerCtx` 包含 timer 和 deadline。
- 知道超时后错误是 `DeadlineExceeded`。
- 解释为什么 `defer cancel()` 能提前释放 timer。

---

## 六、timerCtx 和 cancelCtx 的关系

可以把 `timerCtx` 理解成增强版的 `cancelCtx`。

它不仅能被手动取消，还能被时间自动取消。

心智模型：

```text
timerCtx
  包含 cancelCtx 的取消能力。
  额外保存 deadline。
  额外保存 timer。
```

所以 `WithTimeout` 创建出来的 context，同时具备：

- 父取消时跟着取消。
- 手动 cancel 时取消。
- 时间到了自动取消。

---

## 七、运行实验：提前 cancel

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)

	go func() {
		<-ctx.Done()
		fmt.Println("done:", ctx.Err())
	}()

	time.Sleep(500 * time.Millisecond)
	cancel()
	time.Sleep(200 * time.Millisecond)
}
```

输出是：

```text
done: context canceled
```

虽然设置了 5 秒超时，但你提前调用了 cancel，所以错误原因是 `context.Canceled`。

---

## 八、运行实验：自然超时

```go
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel()

<-ctx.Done()
fmt.Println(ctx.Err())
```

输出是：

```text
context deadline exceeded
```

这说明 timer 到期后触发了取消逻辑，只是错误原因不同。

---

## 九、为什么父 deadline 更早时不创建更晚的 timer

如果父 context 1 秒后超时，子 context 请求 10 秒超时：

```go
parent, parentCancel := context.WithTimeout(context.Background(), time.Second)
defer parentCancel()

child, childCancel := context.WithTimeout(parent, 10*time.Second)
defer childCancel()
```

child 的有效 deadline 不可能晚于 parent。

这符合请求链路：

```text
上游最多给你 1 秒。
你不能要求下游活 10 秒。
```

这也是设计超时预算时必须从入口开始的原因。

---

## 十、常见误解

### 1. WithTimeout 会强制杀死函数？

不会。它只是关闭 `Done()` channel。

函数必须监听：

```go
case <-ctx.Done():
	return ctx.Err()
```

### 2. 超时了就不用 cancel？

仍然建议 `defer cancel()`。这是统一习惯，也能在提前返回时释放 timer。

### 3. timeout 越短越好吗？

不是。过短会导致正常请求被误杀。timeout 是稳定性边界，不是性能优化本身。

---

## 十一、复习题

1. `WithTimeout` 和 `WithDeadline` 的底层关系是什么？
2. 手动 cancel 一个 timeout context 后，`Err()` 是什么？
3. 自然超时后，`Err()` 是什么？
4. 为什么子 context 的 deadline 不应该晚于父 context？
