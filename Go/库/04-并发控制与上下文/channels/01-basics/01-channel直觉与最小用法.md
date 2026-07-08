# 1. channel 直觉与最小用法

本节目标：学完后你能写出最小的 channel 发送和接收代码，并知道它在后端任务协作中的基本作用。

简短引入：后端服务经常要同时做几件事，比如处理请求、写日志、调用下游、刷新缓存。channel 可以理解为 goroutine 之间传递消息的小通道，让一个任务把结果交给另一个任务。

## 一、为什么需要它

假设用户注册成功后，我们想异步写一条欢迎日志。最直接的写法是在主流程里直接调用日志函数，但如果日志写入变慢，注册接口也会跟着慢。

channel 的直觉是：一个 goroutine 负责产生任务，另一个 goroutine 负责消费任务。它们不共享一堆变量，而是通过明确的通道交接数据。

```text
能通过传值解决的问题，不要先急着共享内存和加锁。
```

## 二、基本用法

新建临时文件运行即可。

Windows PowerShell：

```powershell
New-Item -ItemType Directory -Force demo-channel | Out-Null
Set-Location demo-channel
go mod init demo-channel
New-Item main.go
```

Linux/macOS：

```bash
mkdir -p demo-channel
cd demo-channel
go mod init demo-channel
touch main.go
```

把下面代码复制到 `main.go`：

```go
package main

import "fmt"

func main() {
	ch := make(chan string)

	go func() {
		ch <- "user:1001 registered"
	}()

	msg := <-ch
	fmt.Println("write audit log:", msg)
}
```

运行：

```powershell
go run .
```

Linux/macOS 同样运行：

```bash
go run .
```

你会看到：

```text
write audit log: user:1001 registered
```

## 三、关键代码结构

`make(chan string)` 创建一个只能传 `string` 的 channel。真实项目中建议传结构体，因为结构体能带上业务字段，例如用户 ID、动作、时间。

`go func() { ... }()` 启动一个 goroutine，模拟后台任务。

`ch <- "..."` 是发送。可以理解为把消息放进通道。

`msg := <-ch` 是接收。可以理解为从通道取出消息。

这里的 channel 没有缓冲，所以发送方和接收方必须碰头。发送方没有人接收会等，接收方没有人发送也会等。

## 四、真实后端场景示例

下面把字符串换成结构体，更接近项目中的审计日志任务。

```go
package main

import (
	"fmt"
	"time"
)

type AuditEvent struct {
	UserID    int64
	Action    string
	CreatedAt time.Time
}

func main() {
	events := make(chan AuditEvent)

	go func() {
		event := <-events
		fmt.Printf("audit user=%d action=%s at=%s\n",
			event.UserID,
			event.Action,
			event.CreatedAt.Format(time.RFC3339),
		)
	}()

	events <- AuditEvent{
		UserID:    1001,
		Action:    "register",
		CreatedAt: time.Now(),
	}
}
```

真实项目中，这个消费者可能会写数据库或日志系统。这里为了聚焦 channel，只用打印模拟。

涉及数据库时要记住：

```text
channel 只负责把任务交给后台处理；SQL 仍然必须使用参数化查询，不能拼接用户输入。
```

## 五、注意点

channel 适合在同一个进程内做 goroutine 协作。如果服务重启，内存里的 channel 数据会丢。订单、支付、库存扣减这类关键任务，不能只放在内存 channel 里。

如果任务必须可靠送达，真实项目中通常会用数据库状态表、消息队列、事务 outbox 等方式保证可恢复。channel 可以作为进程内消费层，但不是持久化系统。

## 六、常见误区

误区一：以为加了 `go` 就一定更快。  
如果后台任务最后还是抢同一个数据库连接池，过多 goroutine 只会让数据库更慢。

误区二：用全局 channel 到处发送。  
全局 channel 会让数据流向不清楚，出了问题很难知道谁发送、谁接收、谁该关闭。

误区三：把 channel 当消息队列。  
channel 没有持久化、确认、重试、死信队列。关键业务不能只靠它。

## 七、本节练习

练习一：把 `AuditEvent` 增加 `IP string` 和 `TraceID string` 字段，模拟一次用户注册审计日志。真实项目中，trace id 常用于把一条请求在多个服务里的日志串起来。

练习二：把消费者里的打印逻辑改成函数 `writeAuditLog(event AuditEvent) error`，先固定返回 `nil`。然后让它在 `UserID == 0` 时返回错误，并在调用处打印错误。这样做是为了养成一个习惯：后台任务也要有错误出口，不能只靠打印。

练习三：思考并写一句注释：如果这个审计事件不能丢，为什么不能只放在内存 channel 里？可以把答案写在代码上方，提醒自己区分“进程内协作”和“可靠投递”。

## 八、本节达标标准

- 能写出 `make(chan T)`、发送、接收的最小示例。
- 能解释 channel 在 goroutine 之间传递值的作用。
- 能说出 channel 适合进程内协作，不适合单独承担关键业务持久化。
- 能把简单字符串改成业务结构体传递。
