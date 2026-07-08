# 7. worker pool 控制并发处理任务

本节目标：学完后你能用 worker pool 控制并发数量，避免任务把数据库或下游服务打满。

简短引入：后端批量任务常见需求是“快一点处理很多数据”。但最快不等于无限并发。worker pool 的核心是固定数量的 worker 从 channel 取任务。

## 一、为什么需要它

假设要给 10000 个用户补发站内信。如果每个用户一个 goroutine，短时间内可能同时打满数据库连接池、Redis 或短信接口。

worker pool 可以理解为固定数量的工人排队处理任务。任务多了就排队，不会无限扩张。

```text
并发数应该由下游承载能力决定，而不是由任务数量决定。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	jobs := make(chan int)
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		workerID := i
		wg.Add(1)
		go func() {
			defer wg.Done()
			for job := range jobs {
				fmt.Printf("worker %d handle job %d\n", workerID, job)
			}
		}()
	}

	for j := 1; j <= 10; j++ {
		jobs <- j
	}
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

## 三、关键代码结构

`jobs := make(chan int)` 是任务入口。

`for i := 1; i <= 3; i++` 固定启动 3 个 worker。

`for job := range jobs` 让 worker 一直取任务，直到 channel 关闭。

`wg.Wait()` 保证所有 worker 退出后主流程再结束。

## 四、真实后端场景示例

下面模拟批量重算用户等级。真实项目中这类任务通常会读写数据库，因此并发数必须受控。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type UserJob struct {
	UserID int64
}

func recalculateLevel(job UserJob) error {
	time.Sleep(100 * time.Millisecond)
	fmt.Println("recalculate user level:", job.UserID)
	return nil
}

func main() {
	const workerCount = 3
	jobs := make(chan UserJob)
	var wg sync.WaitGroup

	for i := 0; i < workerCount; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for job := range jobs {
				if err := recalculateLevel(job); err != nil {
					fmt.Println("handle job failed:", err)
				}
			}
		}()
	}

	for userID := int64(1001); userID <= 1010; userID++ {
		jobs <- UserJob{UserID: userID}
	}
	close(jobs)

	wg.Wait()
	fmt.Println("batch finished")
}
```

真实项目中，`recalculateLevel` 如果要更新数据库，建议每个用户一笔短事务，失败可重试，不要把所有用户放进一个超大事务。

## 五、注意点

worker 数量要和资源匹配。比如数据库连接池最大 20，不代表 worker 就可以开 20。还要留连接给在线请求，并考虑慢查询、锁等待和索引成本。

任务来源如果是数据库分页，注意不要一次性把所有 ID 读入内存。生产环境常见做法是分批扫描，逐批投递，并记录进度。

涉及迁移任务时要可回滚。比如先新增字段、双写、回填、校验，再切读路径，不要在 worker pool 里直接做不可逆大改。

## 六、常见误区

误区一：任务数多少就开多少 goroutine。  
这会让压力失控，尤其是有数据库写入时。

误区二：worker 里只打印错误。  
批量任务要统计成功、失败、可重试数量，必要时记录失败任务，方便补偿。

误区三：没有关闭 jobs。  
worker 会一直等任务，`wg.Wait()` 永远不返回。

## 七、本节练习

练习一：把 `workerCount` 分别改成 `1`、`3`、`10`，观察批量任务完成速度。然后写一句总结：为什么真实项目不能只看速度，还要看数据库和下游承载能力。

练习二：让 `recalculateLevel` 在某些用户上返回错误，例如 `UserID%3 == 0` 时失败。统计成功数和失败数，而不是只打印错误。

练习三：把任务数量扩大到 `1000`，但不要一次性开 `1000` 个 goroutine。保持 worker pool 模式，并思考如果用户 ID 来自数据库分页，应该如何分批读取和记录进度。

## 八、本节达标标准

- 能写出固定 worker 数量的 worker pool。
- 能解释为什么 worker 数要受数据库和下游能力限制。
- 能用 `close(jobs)` 和 `WaitGroup` 让 worker 正常退出。
- 能说出批量任务要考虑失败记录、重试和可回滚。
