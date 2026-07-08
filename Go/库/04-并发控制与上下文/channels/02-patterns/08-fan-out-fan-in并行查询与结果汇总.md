# 8. fan-out/fan-in 并行查询与结果汇总

本节目标：学完后你能把一个请求拆成多个并行任务，并把结果安全汇总回来。

简短引入：很多后端接口需要组装页面数据，比如用户资料、订单列表、权限信息、通知数量。互不依赖的查询可以并行做，但必须能收集结果和错误。

## 一、为什么需要它

fan-out 是把任务分发出去，fan-in 是把结果收回来。可以理解为一个接口同时找几个同事取资料，最后由你统一整理响应。

这类模式能减少接口总耗时，但它也会增加数据库和下游并发压力。不是所有查询都适合并行。

```text
并行查询只适合互不依赖、耗时相近、下游能承受的任务。
```

## 二、基本用法

```go
package main

import "fmt"

func main() {
	results := make(chan string, 2)

	go func() { results <- "user profile" }()
	go func() { results <- "order summary" }()

	first := <-results
	second := <-results

	fmt.Println(first)
	fmt.Println(second)
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

结果 channel 的容量设置为任务数量，可以避免主流程提前返回时发送方卡住。但这不是逃避取消的理由，真实项目仍然要传 context。

接收次数必须和发送次数匹配。任务数量动态变化时，通常配合 `WaitGroup` 关闭结果 channel，然后 `range` 读取。

## 四、真实后端场景示例

下面模拟用户首页接口，同时加载用户资料、订单数量和权限标记。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Part struct {
	Name  string
	Value string
	Err   error
}

func loadProfile(out chan<- Part) {
	time.Sleep(80 * time.Millisecond)
	out <- Part{Name: "profile", Value: "Alice"}
}

func loadOrderCount(out chan<- Part) {
	time.Sleep(120 * time.Millisecond)
	out <- Part{Name: "orders", Value: "5"}
}

func loadPermission(out chan<- Part) {
	time.Sleep(60 * time.Millisecond)
	out <- Part{Name: "permission", Value: "seller"}
}

func main() {
	results := make(chan Part, 3)
	var wg sync.WaitGroup

	tasks := []func(chan<- Part){loadProfile, loadOrderCount, loadPermission}
	for _, task := range tasks {
		wg.Add(1)
		go func(fn func(chan<- Part)) {
			defer wg.Done()
			fn(results)
		}(task)
	}

	go func() {
		wg.Wait()
		close(results)
	}()

	page := map[string]string{}
	for part := range results {
		if part.Err != nil {
			fmt.Println("load failed:", part.Name, part.Err)
			return
		}
		page[part.Name] = part.Value
	}

	fmt.Println("home page:", page)
}
```

真实接口中，如果其中一个关键查询失败，需要决定是整个接口失败，还是降级返回部分数据。例如通知数量失败可以显示 0，但权限查询失败通常不能放行。

## 五、注意点

并行不是免费的。三个查询并行，就意味着数据库可能同时承受三条 SQL。如果每个请求都这样，高峰期连接池会更快耗尽。

要保守评估索引成本。一个单独看起来还可以的查询，在并发放大后可能变成热点慢查询。上线前应检查执行计划、索引选择和超时设置。

如果这些查询共享同一个事务，不能随意并发使用同一个事务对象。事务边界要清楚，读写一致性要求高时，优先保持流程简单。

## 六、常见误区

误区一：所有查询都并行。  
有依赖关系的任务并行没有意义，还可能读到不一致数据。

误区二：只返回第一个成功结果。  
聚合接口通常需要完整数据。提前返回要配合 context 取消其他任务。

误区三：忽略错误等级。  
真实项目中要区分关键错误和可降级错误，不要一概吞掉。

## 七、本节练习

练习一：让 `loadOrderCount` 返回错误，观察当前代码如何处理。然后把错误分为“关键错误”和“可降级错误”，例如权限失败直接返回，通知数量失败用默认值。

练习二：给三个加载函数都增加 `context.Context` 参数，并在主流程设置整体超时。要求主流程超时后，后台查询不要继续无意义地发送结果。

练习三：把结果 map 改成结构体，例如 `HomePage{Profile string, Orders string, Permission string}`。真实项目中，结构体比随意 map 更容易被编译器和测试保护。

## 八、本节达标标准

- 能解释 fan-out 和 fan-in 的直觉。
- 能用 channel 汇总多个 goroutine 的结果。
- 能用 `WaitGroup` 在所有任务结束后关闭结果 channel。
- 能判断哪些查询适合并行，哪些应保持串行。
