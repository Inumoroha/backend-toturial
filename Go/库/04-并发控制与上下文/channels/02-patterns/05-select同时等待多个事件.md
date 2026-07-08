# 5. select 同时等待多个事件

本节目标：学完后你能用 `select` 同时等待结果、超时和退出信号。

简短引入：后端请求经常要等多个事件：数据库结果、缓存结果、下游 HTTP 返回、超时、用户取消。`select` 就是用来在多个 channel 操作之间做选择。

## 一、为什么需要它

如果只写 `<-ch`，程序会一直等这个 channel。真实项目中这很危险。比如调用风控服务时，对方卡住了，你的请求不能无限等待。

`select` 的直觉是：谁先准备好，就先处理谁。

```text
后端接口等待外部资源时必须有超时或取消路径。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	result := make(chan string)

	go func() {
		time.Sleep(200 * time.Millisecond)
		result <- "risk check passed"
	}()

	select {
	case msg := <-result:
		fmt.Println(msg)
	case <-time.After(100 * time.Millisecond):
		fmt.Println("risk check timeout")
	}
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

这里超时时间是 100ms，而后台任务 200ms 才返回，所以会打印超时。

## 三、关键参数/语法/代码结构

`case msg := <-result` 表示等待业务结果。

`case <-time.After(...)` 表示等待超时。它返回一个 channel，到时间后会收到值。

如果多个 case 同时准备好，Go 会随机选择一个。不要依赖 case 的书写顺序表达优先级。

真实项目中，频繁循环里不要反复创建 `time.After`，可能带来额外分配。更常见的是使用 `context.WithTimeout` 或复用 `time.Timer`。

## 四、真实后端场景示例

下面模拟订单创建前的风控校验。接口不能无限等风控服务。

```go
package main

import (
	"fmt"
	"time"
)

type RiskResult struct {
	Passed bool
	Reason string
}

func checkRisk(orderID int64, out chan<- RiskResult) {
	time.Sleep(150 * time.Millisecond)
	out <- RiskResult{Passed: true, Reason: "ok"}
}

func main() {
	riskCh := make(chan RiskResult, 1)
	go checkRisk(9001, riskCh)

	select {
	case risk := <-riskCh:
		if !risk.Passed {
			fmt.Println("reject order:", risk.Reason)
			return
		}
		fmt.Println("create order")
	case <-time.After(100 * time.Millisecond):
		fmt.Println("reject order: risk check timeout")
	}
}
```

这里 `riskCh` 使用容量 1，是为了在主流程超时返回后，后台 goroutine 仍能把结果放进去并退出。否则它可能卡在发送上。

```text
只让主流程超时还不够，也要考虑后台 goroutine 会不会被卡住。
```

## 五、注意点

`select` 只能等待 channel 操作。普通函数调用不会被它中断。要让数据库查询、HTTP 请求真正取消，需要把 `context` 传进去。

涉及数据库时，真实项目里应该使用支持 context 的方法，例如 `QueryContext`、`ExecContext`。同时 SQL 参数要参数化，事务要尽量短。

如果超时后业务仍然可能成功执行，要考虑幂等。例如订单请求超时，后台任务如果继续扣库存，用户重试可能导致重复扣减。这个问题不是 channel 能单独解决的，需要订单号、唯一索引、事务或幂等表配合。

## 六、常见误区

误区一：以为 `select + time.After` 会停止 goroutine。  
不会。它只让当前 goroutine 不再等。后台 goroutine 是否停止，要看你有没有设计取消机制。

误区二：在业务里滥用 `default`。  
`default` 会让 select 变成非阻塞。用不好会造成空转，占用 CPU。

误区三：忘记超时后的结果发送。  
主流程走了，后台 goroutine 还在向无缓冲 channel 发送，就会泄漏。

## 七、本节练习

练习一：把风控示例里的超时时间从 `100ms` 改成 `300ms`，观察订单是否能创建。这个练习是为了理解：超时设置会直接影响业务结果。

练习二：把 `riskCh := make(chan RiskResult, 1)` 改成无缓冲 channel，再思考主流程超时后后台 goroutine 可能卡在哪里。真实项目中，这类泄漏很隐蔽。

练习三：给风控结果增加 `Err error` 字段，模拟风控服务返回错误。要求区分三种输出：通过、拒绝、超时或错误。后端接口不能把所有失败都混成一个原因。

## 八、本节达标标准

- 能用 `select` 等待业务结果和超时。
- 能解释多个 case 同时就绪时不能依赖顺序。
- 能说出超时不等于取消后台任务。
- 能在可能超时的结果 channel 上合理使用容量 1，避免发送方卡住。
