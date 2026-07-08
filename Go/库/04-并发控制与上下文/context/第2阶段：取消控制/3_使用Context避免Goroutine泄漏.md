# 3. 使用 Context 避免 Goroutine 泄漏

本节目标：理解 goroutine 泄漏的常见原因，并用 `context` 让 goroutine 可退出。

goroutine 泄漏不是内存泄漏那么直观，但在后端服务里同样危险。

泄漏的 goroutine 会占用：

- 内存。
- 调度资源。
- channel。
- 数据库连接。
- 网络连接。
- 锁或其他资源。

---

## 一、一个容易泄漏的例子

```go
func startWorker() {
	go func() {
		for {
			fmt.Println("working")
			time.Sleep(time.Second)
		}
	}()
}
```

这个 worker 没有退出条件。

只要程序不退出，它就一直运行。

如果每个请求都启动一个这样的 goroutine，服务很快就会出问题。

---

## 二、加入 context

```go
func startWorker(ctx context.Context) {
	go func() {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("worker stopped:", ctx.Err())
				return
			default:
				fmt.Println("working")
				time.Sleep(time.Second)
			}
		}
	}()
}
```

现在调用方可以通过取消 context 让 worker 退出。

---

## 三、default 分支要小心

下面写法容易导致 CPU 空转：

```go
for {
	select {
	case <-ctx.Done():
		return
	default:
	}
}
```

因为 `default` 会立即执行，循环会疯狂空转。

如果使用 `default`，里面通常要有真实工作或等待：

```go
default:
	doWork()
	time.Sleep(100 * time.Millisecond)
```

很多场景也可以不用 `default`：

```go
for {
	select {
	case <-ctx.Done():
		return
	case job := <-jobs:
		handle(job)
	}
}
```

---

## 四、发送结果时也要考虑取消

看这个函数：

```go
func query(resultCh chan<- string) {
	time.Sleep(2 * time.Second)
	resultCh <- "result"
}
```

如果调用方已经超时不接收了，发送可能阻塞。

可以改成：

```go
func query(ctx context.Context, resultCh chan<- string) {
	time.Sleep(2 * time.Second)

	select {
	case resultCh <- "result":
	case <-ctx.Done():
		return
	}
}
```

这样发送结果时也尊重取消。

---

## 五、真实项目中的泄漏来源

常见泄漏来源包括：

- 后台轮询没有退出条件。
- 请求中启动 goroutine，但请求结束后 goroutine 还在跑。
- channel 发送无人接收。
- channel 接收永远等不到发送方。
- 外部调用没有超时。
- worker 队列关闭时没有通知消费者退出。

这些都可以通过 context、channel close、WaitGroup 等机制治理。

---

## 六、本节练习

请完成：

1. 写一个无限循环 worker。
2. 给它加入 `ctx.Done()` 退出逻辑。
3. 启动 3 个 worker。
4. 2 秒后取消 context。
5. 确认每个 worker 都打印退出信息。

进阶：

使用 `sync.WaitGroup` 等待所有 worker 退出。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 识别没有退出条件的 goroutine。
- 在循环里监听 `ctx.Done()`。
- 知道 `default` 分支可能导致空转。
- 在发送 channel 时也考虑取消。
- 能写出不会明显泄漏的 worker。

---

## 八、泄漏示例：发送无人接收

```go
func query() <-chan string {
	ch := make(chan string)
	go func() {
		time.Sleep(2 * time.Second)
		ch <- "result"
	}()
	return ch
}
```

调用方：

```go
select {
case result := <-query():
	fmt.Println(result)
case <-time.After(time.Second):
	fmt.Println("timeout")
}
```

调用方 1 秒后超时，但 goroutine 2 秒后发送结果时可能阻塞。

---

## 九、修复：发送时监听 ctx

```go
func query(ctx context.Context) <-chan string {
	ch := make(chan string)
	go func() {
		defer close(ch)

		select {
		case <-time.After(2 * time.Second):
		case <-ctx.Done():
			return
		}

		select {
		case ch <- "result":
		case <-ctx.Done():
			return
		}
	}()
	return ch
}
```

这里有两个取消点：

```text
等待查询期间。
发送结果期间。
```

---

## 十、如何观察 goroutine 泄漏

学习阶段可以打印：

```go
fmt.Println("goroutines:", runtime.NumGoroutine())
```

真实项目可以使用 pprof：

```text
/debug/pprof/goroutine?debug=2
```

如果 goroutine 数量随请求持续上涨，说明可能有泄漏。

---

## 十一、常见泄漏模式

- for 循环没有退出条件。
- channel 接收永远等不到发送。
- channel 发送无人接收。
- ticker 没有 Stop。
- 外部请求没有 timeout。
- worker 不监听关闭信号。

context 主要解决其中“生命周期通知”的问题。

---

## 十二、本节练习

写一个会泄漏的版本，再写一个修复版本。

要求：

1. 泄漏版：调用方超时后，子 goroutine 卡在发送。
2. 修复版：子 goroutine 监听 ctx，超时后退出。
3. 使用 `runtime.NumGoroutine()` 对比。
