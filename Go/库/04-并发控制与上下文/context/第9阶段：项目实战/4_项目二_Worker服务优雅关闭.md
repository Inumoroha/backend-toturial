# 4. 项目二：Worker 服务优雅关闭

本节目标：实现一个能响应退出信号的后台 worker 服务。

---

## 一、项目目标

实现：

```text
程序启动后持续处理任务。
收到 Ctrl+C 或 SIGTERM 后停止接收新任务。
正在执行的任务最多等待 5 秒。
最后退出程序。
```

---

## 二、根 Context

```go
rootCtx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
```

这个 rootCtx 表示整个进程生命周期。

收到退出信号时，它会取消。

---

## 三、Worker

```go
func worker(ctx context.Context, id int, jobs <-chan int) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("worker stopped:", id)
			return
		case job, ok := <-jobs:
			if !ok {
				fmt.Println("worker jobs closed:", id)
				return
			}
			handleJob(ctx, id, job)
		}
	}
}
```

---

## 四、任务处理也要支持 ctx

```go
func handleJob(ctx context.Context, workerID int, job int) {
	select {
	case <-time.After(time.Second):
		fmt.Printf("worker=%d job=%d done\n", workerID, job)
	case <-ctx.Done():
		fmt.Printf("worker=%d job=%d canceled\n", workerID, job)
	}
}
```

只让外层 worker 监听 ctx 不够。

如果单个任务本身很慢，也要能取消。

---

## 五、关闭流程

```go
rootCtx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

jobs := make(chan int)

for i := 1; i <= 3; i++ {
	go worker(rootCtx, i, jobs)
}

go produceJobs(rootCtx, jobs)

<-rootCtx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

_ = shutdownCtx
fmt.Println("shutdown finished")
```

如果使用 WaitGroup，可以在 shutdownCtx 超时前等待 worker 退出。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 使用 `signal.NotifyContext` 创建进程根 ctx。
- worker 循环监听 ctx。
- 单个任务处理也监听 ctx。
- 理解优雅关闭需要单独的 shutdown timeout。

---

## 七、完整 Worker 示例

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func worker(ctx context.Context, id int, jobs <-chan int, wg *sync.WaitGroup) {
	defer wg.Done()

	for {
		select {
		case <-ctx.Done():
			fmt.Println("worker stopped:", id)
			return
		case job, ok := <-jobs:
			if !ok {
				fmt.Println("jobs closed:", id)
				return
			}
			handleJob(ctx, id, job)
		}
	}
}

func handleJob(ctx context.Context, workerID int, job int) {
	select {
	case <-time.After(time.Second):
		fmt.Printf("worker=%d job=%d done\n", workerID, job)
	case <-ctx.Done():
		fmt.Printf("worker=%d job=%d canceled\n", workerID, job)
	}
}

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	jobs := make(chan int)
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(ctx, i, jobs, &wg)
	}

	go func() {
		defer close(jobs)
		for i := 1; ; i++ {
			select {
			case <-ctx.Done():
				return
			case jobs <- i:
				time.Sleep(300 * time.Millisecond)
			}
		}
	}()

	<-ctx.Done()
	fmt.Println("shutdown signal received")

	done := make(chan struct{})
	go func() {
		wg.Wait()
		close(done)
	}()

	select {
	case <-done:
		fmt.Println("all workers stopped")
	case <-time.After(5 * time.Second):
		fmt.Println("shutdown timeout")
	}
}
```

运行后按 Ctrl+C，观察 worker 是否退出。

---

## 八、关闭流程解释

这个程序有三层生命周期：

```text
进程生命周期：signal ctx。
worker 生命周期：监听 signal ctx。
单个 job 生命周期：handleJob 监听 signal ctx。
```

收到退出信号时：

```text
ctx.Done 关闭。
producer 停止发送任务并关闭 jobs。
worker 收到 ctx 或 jobs close 后退出。
main 等待 worker 退出，最多 5 秒。
```

---

## 九、真实项目中还要考虑什么

真实 worker 通常还要处理：

- 任务是否允许中断。
- 中断后任务是否需要重试。
- 是否需要把任务状态标记为失败。
- 是否需要提交 offset 或 ack 消息。
- 是否需要关闭数据库、Redis、消息队列连接。

context 只负责传递取消信号，业务一致性仍然要自己设计。

---

## 十、常见错误

### 1. 收到信号后直接退出

任务可能处理到一半，状态不一致。

### 2. worker 不监听 ctx

主程序想退出，但 worker 还在跑。

### 3. 任务处理函数不监听 ctx

worker 外层能退出，但当前长任务卡住。

### 4. 没有关闭 jobs channel

worker 可能一直等待新任务。

---

## 十一、本节练习

请扩展：

1. 让某个 job 处理 10 秒。
2. Ctrl+C 后最多等待 5 秒。
3. 超时后打印 `shutdown timeout`。
4. 思考真实系统中这个 job 应该如何补偿。
