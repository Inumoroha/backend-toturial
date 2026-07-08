# 4. close 与 range 让消费者正常退出

本节目标：学完后你能正确关闭 channel，并让消费者用 `range` 自然退出。

简短引入：后端服务不能只会启动 goroutine，还要能让它们停下来。`close` 告诉消费者：不会再有新数据了，处理完已有数据就退出。

## 一、为什么需要它

后台任务常见写法是循环读取 channel。如果没有退出条件，这个 goroutine 会一直等着。服务停止、测试结束、迁移任务完成时，它都可能卡住。

`close` 的直觉是：发送方宣布“任务发完了”。消费者收到这个信号后，不再等待新任务。

```text
通常由发送方关闭 channel；接收方不要随手关闭自己没有所有权的 channel。
```

## 二、基本用法

```go
package main

import "fmt"

func main() {
	jobs := make(chan int)

	go func() {
		for job := range jobs {
			fmt.Println("handle job", job)
		}
		fmt.Println("worker exit")
	}()

	jobs <- 1
	jobs <- 2
	close(jobs)
}
```

为了让输出更稳定，实际运行时可以加一个等待：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	jobs := make(chan int)
	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		for job := range jobs {
			fmt.Println("handle job", job)
		}
		fmt.Println("worker exit")
	}()

	jobs <- 1
	jobs <- 2
	close(jobs)
	wg.Wait()
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

## 三、关键参数/语法/代码结构

`close(jobs)` 表示不会再向 `jobs` 发送新数据。

`for job := range jobs` 会不断接收数据，直到 channel 被关闭且数据被取完。

`sync.WaitGroup` 用于等待 goroutine 完成。它和 channel 不是竞争关系，真实项目中经常一起使用。

## 四、真实后端场景示例

下面模拟文章索引重建任务。主流程把文章 ID 发给 worker，发完后关闭 channel，worker 处理完自然退出。

```go
package main

import (
	"fmt"
	"sync"
)

func rebuildSearchIndex(articleID int64) {
	fmt.Println("rebuild article index:", articleID)
}

func main() {
	articleIDs := []int64{101, 102, 103}
	jobs := make(chan int64)
	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		for id := range jobs {
			rebuildSearchIndex(id)
		}
		fmt.Println("index worker stopped")
	}()

	for _, id := range articleIDs {
		jobs <- id
	}
	close(jobs)

	wg.Wait()
	fmt.Println("all done")
}
```

真实项目中，如果重建索引会写数据库或搜索引擎，要注意每个任务失败后的重试策略。不要在一个大事务里处理所有文章，否则失败回滚成本很高。

## 五、注意点

关闭 channel 后不能再发送，否则会 panic。

从已关闭 channel 接收会立即返回零值。如果你不用 `range`，而是手动接收，可以用第二个返回值判断是否关闭：

```go
job, ok := <-jobs
if !ok {
	return
}
fmt.Println(job)
```

多个发送方同时存在时，关闭会变难。常见做法是由一个协调者等所有发送方结束后统一关闭。

## 六、常见误区

误区一：接收方关闭 channel。  
如果还有发送方继续发送，会直接 panic。除非接收方明确拥有这个 channel，否则不要关闭。

误区二：以为关闭 channel 会清空数据。  
不会。已进入缓冲的数据仍然可以被 `range` 取完。

误区三：用 `time.Sleep` 等 goroutine 结束。  
Sleep 只是碰运气。真实项目和测试里应该用 `WaitGroup`、`context` 或明确的完成信号。

## 七、本节练习

练习一：在文章索引重建示例里增加第二个 worker，观察两个 worker 如何共同消费同一个 `jobs` channel。真实项目中，这就是 worker pool 的雏形。

练习二：故意把 `close(jobs)` 注释掉，观察 `wg.Wait()` 是否能返回。这个练习用来理解：消费者使用 `range` 时，关闭 channel 是退出信号。

练习三：在 `rebuildSearchIndex` 中模拟某个文章 ID 失败，例如 `id == 102` 时返回错误。先打印错误，再思考：如果这是批量重建搜索索引，失败 ID 应该记录到哪里，后续如何重试？

## 八、本节达标标准

- 能用 `close` 和 `range` 让消费者退出。
- 能说明为什么通常由发送方关闭 channel。
- 能用 `WaitGroup` 等待后台 goroutine 完成。
- 能识别关闭后继续发送导致 panic 的风险。
