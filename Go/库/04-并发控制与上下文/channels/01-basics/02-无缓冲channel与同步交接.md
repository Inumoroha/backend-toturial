# 2. 无缓冲 channel 与同步交接

本节目标：学完后你能解释无缓冲 channel 为什么会阻塞，并能把它用于需要明确交接时机的后端任务。

简短引入：无缓冲 channel 可以理解为“当面交接”。发送方必须等接收方接住，接收方也必须等发送方交出数据。

## 一、为什么需要它

后端里有些事情不能只“丢出去就不管”。例如迁移任务启动前，主 goroutine 想确认 worker 已经准备好；或者测试中想确认后台 goroutine 已经处理完一条任务。

无缓冲 channel 的价值不是缓存，而是同步时机。它能让两个 goroutine 在某个点上确定地碰头。

```text
无缓冲 channel 常用于明确交接和同步，不用于削峰填谷。
```

## 二、基本用法

创建 `main.go`：

```go
package main

import "fmt"

func main() {
	done := make(chan struct{})

	go func() {
		fmt.Println("worker: refresh permission cache")
		done <- struct{}{}
	}()

	<-done
	fmt.Println("main: worker finished")
}
```

运行：

Windows PowerShell：

```powershell
go run .
```

Linux/macOS：

```bash
go run .
```

`struct{}{}` 常用于只通知事件，不传递具体数据。它不占业务含义，读起来也清楚：这里只关心“完成了”。

## 三、关键代码结构

`done := make(chan struct{})` 表示创建一个完成通知通道。

`done <- struct{}{}` 表示 worker 通知主流程：任务完成。

`<-done` 表示主流程等待通知。没有收到之前，它会停住。

这个停住不是 bug，而是我们主动要求的同步点。

## 四、真实后端场景示例

下面模拟启动服务时刷新权限缓存。缓存没准备好之前，不应该对外提供依赖权限判断的接口。

```go
package main

import (
	"fmt"
	"time"
)

func loadPermissionCache(done chan struct{}) {
	time.Sleep(500 * time.Millisecond)
	fmt.Println("permission cache loaded")
	done <- struct{}{}
}

func main() {
	ready := make(chan struct{})

	go loadPermissionCache(ready)

	fmt.Println("server: waiting cache")
	<-ready
	fmt.Println("server: start accepting requests")
}
```

真实项目中，缓存加载可能来自数据库。注意数据库查询必须有超时和错误处理，不能无限等。这里先只学习同步交接，后面会用 `context` 处理取消和超时。

## 五、注意点

无缓冲 channel 最容易出现的问题是：只有发送，没有接收；或者只有接收，没有发送。这样程序会一直阻塞。

后端服务里，如果请求处理 goroutine 阻塞在一个没人接收的 channel 上，这个请求就不会返回。并发高时会慢慢拖垮服务。

```text
写无缓冲 channel 前先问三件事：谁发送？谁接收？任何一方出错时谁负责退出？
```

## 六、常见误区

误区一：在同一个 goroutine 里先发送再接收。

```go
ch := make(chan string)
ch <- "hello"
fmt.Println(<-ch)
```

这会死锁，因为发送时还没有接收者。

误区二：把无缓冲 channel 当队列。  
无缓冲 channel 没有队列容量，它强调交接时机。想做临时排队要看有缓冲 channel 或专门队列。

误区三：忽略错误分支。  
如果 worker 读取数据库失败但没有通知 `done`，主流程可能永远等待。真实项目要把错误也返回，不能只返回完成信号。

## 七、本节练习

练习一：把权限缓存示例中的 `time.Sleep(500 * time.Millisecond)` 改成 `2 * time.Second`，观察主流程是否会等待。这个练习是为了让你真正体会无缓冲 channel 的同步交接。

练习二：故意注释掉 `done <- struct{}{}`，再次运行程序，观察主流程会发生什么。真实项目中，这类问题常表现为接口一直不返回或启动流程卡住。

练习三：把 `done chan struct{}` 改成 `result chan error`，让 `loadPermissionCache` 能返回错误。比如模拟数据库失败时发送 `fmt.Errorf("load permission cache failed")`。这一步是在为后面的错误传递模式做铺垫。

## 八、本节达标标准

- 能说明无缓冲 channel 的发送和接收必须同时准备好。
- 能用 `chan struct{}` 表达完成通知。
- 能识别同一个 goroutine 中先发送后接收导致的死锁。
- 能判断一个场景是需要同步交接，还是需要缓冲排队。
