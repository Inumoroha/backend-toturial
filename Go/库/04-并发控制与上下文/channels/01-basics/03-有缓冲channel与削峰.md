# 3. 有缓冲 channel 与削峰

本节目标：学完后你能用有缓冲 channel 做小规模任务缓冲，并知道缓冲大小背后的风险。

简短引入：有缓冲 channel 可以理解为一个有限长度的任务槽。发送方可以先放几条任务进去，消费者稍后再处理。

## 一、为什么需要它

真实后端服务中，请求流量经常不是完全平稳的。比如短链接访问日志、文章浏览记录、异步审计事件，可能在某几秒突然变多。

有缓冲 channel 可以暂时吸收一点波动，让生产者和消费者不必每条都同步碰头。但它的容量必须有限，不能无限扩张。

```text
缓冲不是免费性能，它只是把压力暂时放进内存。
```

## 二、基本用法

```go
package main

import "fmt"

func main() {
	jobs := make(chan string, 2)

	jobs <- "audit:user:1001"
	jobs <- "audit:user:1002"

	fmt.Println(<-jobs)
	fmt.Println(<-jobs)
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

容量是 2，所以前两次发送可以直接进入缓冲。再发送第三条时，如果没有消费者取走，就会阻塞。

## 三、关键参数/语法/代码结构

`make(chan string, 2)` 的第二个参数就是缓冲容量。

`len(jobs)` 可以看当前缓冲里有多少条数据。

`cap(jobs)` 可以看最大容量。

这些值适合做调试或指标，不适合拿来做复杂业务判断。因为并发环境下，刚读完 `len`，另一个 goroutine 可能已经发送或接收了。

## 四、真实后端场景示例

下面模拟短链接访问日志。请求处理方只把日志事件放入 channel，后台消费者慢慢写。

```go
package main

import (
	"fmt"
	"time"
)

type VisitLog struct {
	ShortCode string
	UserID    int64
}

func main() {
	logs := make(chan VisitLog, 3)

	go func() {
		for log := range logs {
			time.Sleep(200 * time.Millisecond)
			fmt.Printf("write visit log: code=%s user=%d\n", log.ShortCode, log.UserID)
		}
	}()

	for i := 1; i <= 5; i++ {
		logs <- VisitLog{ShortCode: "go123", UserID: int64(1000 + i)}
		fmt.Println("accepted log", i)
	}

	close(logs)
	time.Sleep(time.Second)
}
```

这个例子能看到生产者可能先连续接受几条日志，但当消费者处理不过来时，生产者仍然会被阻塞。

真实项目中，如果访问日志可以少量丢弃，可能会选择非阻塞发送；如果审计日志必须完整，应该写数据库或可靠队列，不要只靠内存缓冲。

## 五、注意点

缓冲大小没有通用答案。常见判断方式是：

- 请求能不能等待。
- 任务能不能丢。
- 消费者平均处理速度是多少。
- 内存能承受多少积压。
- 下游数据库或日志系统是否有连接数限制。

如果缓冲满了，真实项目通常有三种选择：等待、返回错误、丢弃低价值任务。不要悄悄开更多 goroutine 硬顶。

## 六、常见误区

误区一：缓冲越大越好。  
缓冲越大，故障时积压越多，服务退出也越慢，还可能让问题更晚暴露。

误区二：把缓冲当可靠存储。  
进程崩溃后，channel 里的数据会消失。订单、支付、库存流水不能只放在 channel 中。

误区三：满了以后无限新建 goroutine。  
这相当于把 channel 的边界绕开了，最终压力会转移到数据库连接池、内存或调度器。

## 七、本节练习

练习一：把访问日志示例中的缓冲容量从 `3` 分别改成 `1`、`5`、`10`，观察 `accepted log` 的打印速度。你要重点看的是：缓冲只能吸收短暂波动，消费者慢时生产者最终仍会被拖住。

练习二：把消费者的 `time.Sleep(200 * time.Millisecond)` 改成 `1 * time.Second`，模拟日志系统变慢。然后思考：在线请求是应该等待、返回错误，还是丢弃这类日志？

练习三：尝试用 `select` 写一个非阻塞发送：当 `logs` 满了时打印 `drop visit log`。只建议对低价值统计事件这样做，审计、订单、资金流水不要这样丢。

## 八、本节达标标准

- 能写出 `make(chan T, n)` 的有缓冲 channel。
- 能解释容量满时发送会阻塞。
- 能根据任务重要性判断满了以后等待、返回错误还是丢弃。
- 能说出有缓冲 channel 不能替代消息队列。
