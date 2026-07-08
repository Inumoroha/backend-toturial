# 1. WithCancel 基础用法

本节目标：掌握 `context.WithCancel` 的基本写法和使用场景。

基本形式：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()
```

`WithCancel` 会基于父 context 创建一个新的子 context，并返回一个取消函数。

---

## 一、最小示例

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	go func() {
		<-ctx.Done()
		fmt.Println("context canceled:", ctx.Err())
	}()

	time.Sleep(time.Second)
	cancel()
	time.Sleep(500 * time.Millisecond)
}
```

运行后会看到：

```text
context canceled: context canceled
```

`ctx.Err()` 返回的是：

```go
context.Canceled
```

---

## 二、为什么要 defer cancel

常见写法：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()
```

即使你觉得函数会自然结束，也建议调用 `cancel()`。

原因是：

- 释放 context 内部资源。
- 通知子 context 和监听者退出。
- 形成统一习惯，避免漏掉。

尤其是 `WithTimeout` 和 `WithDeadline`，不调用 `cancel()` 会让内部定时器资源更晚释放。

---

## 三、worker 示例

```go
func worker(ctx context.Context, id int) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("worker %d stopped: %v\n", id, ctx.Err())
			return
		default:
			fmt.Printf("worker %d working\n", id)
			time.Sleep(300 * time.Millisecond)
		}
	}
}
```

主函数：

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	for i := 1; i <= 3; i++ {
		go worker(ctx, i)
	}

	time.Sleep(time.Second)
	cancel()
	time.Sleep(time.Second)
}
```

这里 3 个 worker 使用同一个 ctx。

调用一次 `cancel()`，它们都会收到信号。

---

## 四、什么时候不该新建 WithCancel

不要在每一层都无意义地创建可取消 context。

不推荐：

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	return s.repo.FindByID(ctx, id)
}
```

如果 service 没有新的取消边界，不需要派生新 context。

推荐：

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	return s.repo.FindByID(ctx, id)
}
```

只有当你确实需要控制子任务生命周期时，再使用 `WithCancel`。

---

## 五、本节练习

请完成：

1. 启动 5 个 worker。
2. 每个 worker 每 200ms 打印一次自己的 id。
3. 主 goroutine 运行 2 秒后调用 `cancel()`。
4. 每个 worker 收到取消后打印退出信息。
5. 确认所有 worker 都能退出。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 写出 `ctx, cancel := context.WithCancel(parent)`。
- 知道调用 `cancel()` 会关闭 `ctx.Done()`。
- 在 worker 中监听 `ctx.Done()`。
- 知道不是每一层都需要创建新的 cancel context。

---

## 七、完整 WaitGroup 示例

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func worker(ctx context.Context, id int, wg *sync.WaitGroup) {
	defer wg.Done()

	for {
		select {
		case <-ctx.Done():
			fmt.Printf("worker %d stopped: %v\n", id, ctx.Err())
			return
		case <-time.After(300 * time.Millisecond):
			fmt.Printf("worker %d working\n", id)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	var wg sync.WaitGroup
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(ctx, i, &wg)
	}

	time.Sleep(time.Second)
	cancel()
	wg.Wait()
	fmt.Println("all workers stopped")
}
```

运行后可以看到所有 worker 都退出，main 最后结束。

---

## 八、为什么这里不用 defer cancel

在这个 main 中，cancel 是业务动作：

```go
time.Sleep(time.Second)
cancel()
```

也可以加：

```go
defer cancel()
```

防止未来代码提前返回时漏掉取消。

在普通函数中，创建可取消 context 后建议立刻写 defer：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()
```

---

## 九、常见错误

### 1. worker 使用 default 忙等

```go
for {
	select {
	case <-ctx.Done():
		return
	default:
	}
}
```

这会空转 CPU。

### 2. cancel 后不等待 worker

main 可能提前退出，日志还没打印完。

### 3. 任务函数吞掉取消错误

取消时应返回或记录 `ctx.Err()`。

---

## 十、本节练习

把完整示例改成：

1. 5 个 worker。
2. 每个 worker 处理 job channel。
3. main 发送 10 个 job。
4. 发送到一半时 cancel。
5. worker 能退出，main 能等待。
