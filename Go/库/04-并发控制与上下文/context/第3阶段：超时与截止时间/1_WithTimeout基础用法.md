# 1. WithTimeout 基础用法

本节目标：掌握 `context.WithTimeout` 的基础写法和行为。

基本形式：

```go
ctx, cancel := context.WithTimeout(parent, duration)
defer cancel()
```

当超过 `duration` 后，context 会自动取消，`ctx.Err()` 返回：

```go
context.DeadlineExceeded
```

---

## 一、最小示例

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func work(ctx context.Context) error {
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
		fmt.Println("work timeout")
		return
	}
	if err != nil {
		fmt.Println("work error:", err)
		return
	}

	fmt.Println("success")
}
```

任务需要 2 秒，context 只有 1 秒，所以会超时。

---

## 二、超时本质上也是取消

不管是主动调用 `cancel()`，还是超时时间到了，`ctx.Done()` 都会关闭。

区别在于 `ctx.Err()`：

```text
主动 cancel：context.Canceled
超时：context.DeadlineExceeded
```

所以函数内部通常不用关心是哪一种：

```go
case <-ctx.Done():
	return ctx.Err()
```

由调用方决定如何处理。

---

## 三、为什么 defer cancel

下面写法不完整：

```go
ctx, _ := context.WithTimeout(parent, time.Second)
```

推荐：

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()
```

`cancel()` 不只是“手动取消”。

它还用于释放内部资源。尤其是在任务提前完成时，调用 `cancel()` 可以让定时器不必等到超时时间自然到达。

---

## 四、任务提前完成

如果任务 500ms 完成，timeout 是 2 秒：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

err := fastWork(ctx)
```

函数返回时执行 `defer cancel()`，释放资源。

这就是为什么即使没有超时，也要 cancel。

---

## 五、练习

请写一个函数：

```go
func simulate(ctx context.Context, cost time.Duration) error
```

要求：

1. `cost` 表示任务耗时。
2. 使用 `select` 同时等待 `time.After(cost)` 和 `ctx.Done()`。
3. main 中分别使用 1 秒、3 秒 timeout 调用。
4. 用 `errors.Is` 判断是否 `DeadlineExceeded`。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 使用 `context.WithTimeout` 创建超时 context。
- 取消时返回 `ctx.Err()`。
- 用 `errors.Is` 判断超时。
- 解释为什么要 `defer cancel()`。

---

## 七、完整运行版本

可以创建：

```text
cmd/03-timeout-basic/main.go
```

代码：

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func simulate(ctx context.Context, name string, cost time.Duration) error {
	fmt.Println(name, "start")

	select {
	case <-time.After(cost):
		fmt.Println(name, "done")
		return nil
	case <-ctx.Done():
		return fmt.Errorf("%s stopped: %w", name, ctx.Err())
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	err := simulate(ctx, "job", 2*time.Second)
	if errors.Is(err, context.DeadlineExceeded) {
		fmt.Println("main: timeout")
		return
	}
	if err != nil {
		fmt.Println("main:", err)
		return
	}

	fmt.Println("main: success")
}
```

运行：

```powershell
go run ./cmd/03-timeout-basic
```

你会看到任务被超时停止。

---

## 八、修改参数观察

把任务耗时改成：

```go
500 * time.Millisecond
```

任务会成功。

把 timeout 改成：

```go
10 * time.Millisecond
```

几乎一定超时。

通过反复修改参数，你会理解：

```text
timeout 是调用方给任务设置的时间边界。
任务能否及时退出，取决于它是否监听 ctx.Done。
```

---

## 九、常见错误

### 1. 使用 time.Sleep 模拟任务但不监听 ctx

```go
time.Sleep(2 * time.Second)
return nil
```

这种任务不会被 context 打断。

更推荐用：

```go
select {
case <-time.After(2 * time.Second):
case <-ctx.Done():
}
```

### 2. 返回普通错误

```go
return errors.New("timeout")
```

这样调用方无法用 `errors.Is` 判断 `DeadlineExceeded`。

### 3. timeout 写死在底层

底层可以设置子 timeout，但要基于上游 ctx，不要完全脱离调用链。

---

## 十、本节加餐练习

写一个 `upload(ctx, file string, cost time.Duration)`：

1. 开始时打印文件名。
2. 成功时打印上传完成。
3. 超时时返回包装后的 `ctx.Err()`。
4. main 中分别测试成功和超时。
5. 用 `errors.Is` 判断结果。
