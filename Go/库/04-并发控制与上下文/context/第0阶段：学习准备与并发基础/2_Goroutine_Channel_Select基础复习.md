# 2. Goroutine、Channel、Select 基础复习

本节目标：复习 `context` 经常依赖的三个并发基础：goroutine、channel、select。

`context` 的取消信号最终也是通过 channel 暴露出来的：

```go
ctx.Done()
```

所以在学习 `context` 之前，必须知道 channel 和 select 的基本用法。

---

## 一、goroutine 是什么

goroutine 是 Go 中非常轻量的并发执行单元。

启动方式：

```go
go func() {
	fmt.Println("hello from goroutine")
}()
```

完整示例：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	go func() {
		fmt.Println("hello from goroutine")
	}()

	time.Sleep(time.Second)
	fmt.Println("main done")
}
```

这里 `main` 函数如果直接结束，整个程序也会退出，所以示例中用 `time.Sleep` 等待了一下。

真实项目里通常不会用 `Sleep` 等 goroutine，而是用 channel、WaitGroup 或 context 管理生命周期。

---

## 二、channel 是什么

channel 用来在 goroutine 之间传递数据或信号。

示例：

```go
package main

import "fmt"

func main() {
	ch := make(chan string)

	go func() {
		ch <- "result"
	}()

	result := <-ch
	fmt.Println(result)
}
```

这段代码表达的是：

```text
子 goroutine 计算结果。
主 goroutine 等待结果。
结果通过 channel 传回来。
```

---

## 三、channel 也可以只传信号

有时候我们不关心传什么数据，只关心“事件发生了”。

可以使用：

```go
done := make(chan struct{})
```

`struct{}` 是空结构体，不占额外数据空间，适合表示信号。

示例：

```go
done := make(chan struct{})

go func() {
	time.Sleep(time.Second)
	close(done)
}()

<-done
fmt.Println("received done signal")
```

注意这里用的是 `close(done)`。

关闭 channel 后，所有等待接收的地方都会被唤醒。

这和 `ctx.Done()` 的行为很像：当 context 被取消时，`Done()` 返回的 channel 会被关闭。

---

## 四、select 是什么

`select` 可以同时等待多个 channel。

示例：

```go
select {
case result := <-resultCh:
	fmt.Println("result:", result)
case <-timeoutCh:
	fmt.Println("timeout")
}
```

谁先准备好，就执行谁。

这非常适合表达：

```text
要么任务完成。
要么超时。
要么收到取消信号。
```

---

## 五、模拟一个超时等待

不用 `context`，也可以先用 channel 理解超时模式：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	resultCh := make(chan string)

	go func() {
		time.Sleep(2 * time.Second)
		resultCh <- "work done"
	}()

	select {
	case result := <-resultCh:
		fmt.Println(result)
	case <-time.After(time.Second):
		fmt.Println("timeout")
	}
}
```

因为任务要 2 秒，超时只给 1 秒，所以会输出：

```text
timeout
```

这就是 `context.WithTimeout` 背后的使用场景：我们不想无限等待一个任务。

---

## 六、为什么只超时还不够

上面的示例有一个隐藏问题。

主 goroutine 超时返回了，但子 goroutine 还在继续睡眠。睡完以后，它会尝试发送结果：

```go
resultCh <- "work done"
```

如果没有人接收，它可能阻塞。

在复杂程序中，这类问题会造成 goroutine 泄漏。

所以真实项目里，子 goroutine 也应该知道任务已经取消。

这正是 `context` 的价值：

```text
不仅调用方停止等待。
被调用方也能收到取消信号并退出。
```

---

## 七、本节练习

请完成下面几个练习：

1. 启动一个 goroutine，1 秒后通过 channel 返回字符串。
2. 使用 `select` 同时等待结果和 `time.After`。
3. 把任务耗时改成 500ms，观察结果。
4. 把任务耗时改成 2s，观察超时。
5. 思考：主 goroutine 超时后，子 goroutine 是否还在运行？

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 启动一个 goroutine。
- 使用 channel 传递结果。
- 使用 `close(done)` 传递完成信号。
- 使用 `select` 等待多个 channel。
- 理解“调用方不等了”和“任务真正停止了”不是一回事。

---

## 九、完整练习：结果、超时、取消三选一

下面这个例子非常接近后面 `context` 的写法：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	resultCh := make(chan string)
	cancelCh := make(chan struct{})

	go func() {
		time.Sleep(2 * time.Second)

		select {
		case resultCh <- "result":
		case <-cancelCh:
			fmt.Println("worker canceled before sending result")
		}
	}()

	go func() {
		time.Sleep(500 * time.Millisecond)
		close(cancelCh)
	}()

	select {
	case result := <-resultCh:
		fmt.Println("got:", result)
	case <-cancelCh:
		fmt.Println("main canceled")
	case <-time.After(time.Second):
		fmt.Println("main timeout")
	}
}
```

这个程序有三个可能事件：

```text
任务完成。
主动取消。
等待超时。
```

context 只是把这些模式标准化了。

---

## 十、无缓冲 channel 的阻塞

无缓冲 channel 发送和接收必须同时准备好。

```go
ch := make(chan string)
ch <- "hello"
```

如果没有 goroutine 接收，这行会阻塞。

这也是为什么发送结果时要考虑取消：

```go
select {
case resultCh <- result:
case <-done:
	return
}
```

---

## 十一、close channel 的广播效果

关闭 channel 后，多个接收方都会被唤醒。

```go
done := make(chan struct{})

for i := 0; i < 3; i++ {
	go func(id int) {
		<-done
		fmt.Println("stopped", id)
	}(i)
}

close(done)
```

这和 context 取消很像：一个取消信号可以通知多个 goroutine。

---

## 十二、常见错误

### 1. 在多个地方关闭同一个 channel

重复 close 会 panic。

context 的 cancel 函数是幂等的，帮你处理了这个问题。

### 2. select 里 default 空转

```go
for {
	select {
	default:
	}
}
```

这会消耗 CPU。

### 3. 主 goroutine 退出太快

main 结束后，其他 goroutine 也会结束。

学习 demo 中可以用 channel 或 WaitGroup 等待。

---

## 十三、本节复盘问题

1. channel 传结果和传信号有什么区别？
2. 为什么 `struct{}` 常用于 done channel？
3. `select` 中多个 case 同时就绪时会怎样？
4. 关闭 channel 和发送一个值有什么区别？
5. 为什么调用方超时不代表子 goroutine 已经停止？
