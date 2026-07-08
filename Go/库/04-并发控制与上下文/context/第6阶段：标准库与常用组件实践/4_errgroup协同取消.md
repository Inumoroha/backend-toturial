# 4. errgroup 协同取消

本节目标：学习 `errgroup.WithContext`，处理多个 goroutine 并发执行和错误取消。

安装依赖：

```powershell
go get golang.org/x/sync/errgroup
```

---

## 一、为什么需要 errgroup

`sync.WaitGroup` 只能等待 goroutine 结束。

它不负责：

- 收集错误。
- 任一 goroutine 失败时取消其他 goroutine。
- 传递取消 context。

`errgroup` 可以更方便地处理这些问题。

---

## 二、基础示例

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
	return task(ctx, "a", 100*time.Millisecond)
})

g.Go(func() error {
	return task(ctx, "b", 300*time.Millisecond)
})

if err := g.Wait(); err != nil {
	return err
}
```

如果某个任务返回错误，`errgroup` 会取消派生出来的 ctx。

其他监听 ctx 的任务可以尽快退出。

---

## 三、任务必须尊重 ctx

```go
func task(ctx context.Context, name string, cost time.Duration) error {
	select {
	case <-time.After(cost):
		fmt.Println(name, "done")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

如果 task 完全不监听 `ctx.Done()`，errgroup 取消也没有意义。

---

## 四、聚合接口场景

一个接口并发查询三个下游：

```go
g, ctx := errgroup.WithContext(r.Context())

g.Go(func() error {
	return fetchUser(ctx)
})

g.Go(func() error {
	return fetchOrders(ctx)
})

g.Go(func() error {
	return fetchWallet(ctx)
})

if err := g.Wait(); err != nil {
	http.Error(w, err.Error(), http.StatusBadGateway)
	return
}
```

任意一个返回错误，其他任务会收到取消信号。

---

## 五、errgroup 和 WaitGroup 怎么选

使用 WaitGroup：

```text
只需要等待所有 goroutine 完成。
错误不重要或自己处理。
不需要自动取消。
```

使用 errgroup：

```text
多个任务都可能返回错误。
任一失败要取消其他任务。
需要和 context 配合。
```

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 使用 `errgroup.WithContext`。
- 在多个 goroutine 中共享派生 ctx。
- 任一任务失败后取消其他任务。
- 区分 WaitGroup 和 errgroup 的适用场景。

---

## 七、完整可运行示例

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/sync/errgroup"
)

func task(ctx context.Context, name string, cost time.Duration, fail bool) error {
	select {
	case <-time.After(cost):
		if fail {
			return fmt.Errorf("%s failed", name)
		}
		fmt.Println(name, "done")
		return nil
	case <-ctx.Done():
		return fmt.Errorf("%s stopped: %w", name, ctx.Err())
	}
}

func main() {
	g, ctx := errgroup.WithContext(context.Background())

	g.Go(func() error {
		return task(ctx, "user", 300*time.Millisecond, false)
	})

	g.Go(func() error {
		return task(ctx, "orders", 500*time.Millisecond, true)
	})

	g.Go(func() error {
		return task(ctx, "wallet", 2*time.Second, false)
	})

	if err := g.Wait(); err != nil {
		fmt.Println("group error:", err)
	}
}
```

运行：

```powershell
go run .
```

你会看到 `orders` 失败后，`wallet` 不会继续等满 2 秒，而是收到取消信号。

---

## 八、errgroup 的返回错误

`g.Wait()` 返回第一个非 nil 错误。

如果多个 goroutine 都返回错误，通常只保留其中一个。

所以任务返回错误时要带清楚上下文：

```go
return fmt.Errorf("fetch orders failed: %w", err)
```

否则日志里只有：

```text
context canceled
```

你会不知道是哪个任务触发了取消。

---

## 九、限制并发数量

新版 `errgroup.Group` 支持 `SetLimit`：

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(5)
```

适合处理很多任务时限制并发。

例如批量调用 100 个下游资源，但最多同时跑 5 个。

学习初期先掌握基础并发取消，再学习并发限制。

---

## 十、常见错误

### 1. 闭包变量问题

循环中启动 goroutine 时要小心：

```go
for _, id := range ids {
	id := id
	g.Go(func() error {
		return fetch(ctx, id)
	})
}
```

### 2. 任务不监听 ctx

如果任务里只有：

```go
time.Sleep(10 * time.Second)
```

它不会因为 errgroup 取消而提前停止。

### 3. 在 goroutine 中写共享 map

多个 goroutine 同时写 map 会 data race。

需要加锁，或者每个任务返回结果后统一合并。

---

## 十一、本节练习

请写一个批量查询 demo：

1. 有 10 个 userID。
2. 使用 errgroup 并发查询。
3. 设置并发上限为 3。
4. 任一查询失败后取消其他查询。
5. 每个查询都监听 ctx。
