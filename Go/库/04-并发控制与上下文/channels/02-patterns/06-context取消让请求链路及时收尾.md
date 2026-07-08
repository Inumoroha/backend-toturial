# 6. context 取消让请求链路及时收尾

本节目标：学完后你能把 `context.Context` 和 channel 配合起来，让请求取消或超时时后台任务及时停止。

简短引入：后端请求有生命周期。客户端断开、网关超时、服务关闭时，继续处理已经没人要的任务通常是浪费，甚至会造成脏写。

## 一、为什么需要它

只用 channel 能传结果，但不擅长表达“这个请求不要了”。Go 后端中通常用 `context` 表达取消、超时和请求级元数据。

channel 和 context 的分工可以理解为：

- channel 传业务数据或任务。
- context 传取消信号和截止时间。

```text
请求级 goroutine 必须能感知 context 取消，否则高并发超时会慢慢堆出 goroutine 泄漏。
```

## 二、基本用法

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	done := make(chan string, 1)

	go func() {
		time.Sleep(200 * time.Millisecond)
		done <- "profile loaded"
	}()

	select {
	case msg := <-done:
		fmt.Println(msg)
	case <-ctx.Done():
		fmt.Println("request canceled:", ctx.Err())
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

## 三、关键参数/语法/代码结构

`context.WithTimeout` 创建一个带超时的 context。

`defer cancel()` 释放内部资源。即使自然超时，也建议调用。

`ctx.Done()` 返回一个 channel。取消或超时时，这个 channel 会关闭。

`ctx.Err()` 可以看到取消原因，常见是 `context.Canceled` 或 `context.DeadlineExceeded`。

## 四、真实后端场景示例

下面模拟查询用户资料。worker 在每一步处理前检查 context，发现请求取消就退出。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

type UserProfile struct {
	UserID int64
	Name   string
}

func loadProfile(ctx context.Context, userID int64, out chan<- UserProfile) {
	select {
	case <-time.After(200 * time.Millisecond):
		profile := UserProfile{UserID: userID, Name: "Alice"}
		select {
		case out <- profile:
		case <-ctx.Done():
			return
		}
	case <-ctx.Done():
		return
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	result := make(chan UserProfile, 1)
	go loadProfile(ctx, 1001, result)

	select {
	case profile := <-result:
		fmt.Println("profile:", profile)
	case <-ctx.Done():
		fmt.Println("load profile failed:", ctx.Err())
	}
}
```

真实项目中，如果 `loadProfile` 查询数据库，应该传入 `ctx`：

```go
// db.QueryContext(ctx, "select id, name from users where id = ?", userID)
```

不同数据库驱动占位符不同，例如 PostgreSQL 常用 `$1`。无论哪种写法，都不要拼接用户输入。

## 五、注意点

context 是取消信号，不是万能中断器。你的代码必须主动检查 `<-ctx.Done()`，数据库驱动和 HTTP 客户端也必须支持 context。

不要把 context 存到结构体里长期保存。真实项目中通常从请求入口传到调用链下游。

不要用 context 传大量业务参数。用户 ID、订单 ID 可以作为函数参数；context 适合 trace id、鉴权信息这类横切信息，但也要克制。

## 六、常见误区

误区一：创建了 context 但不传下去。  
这样取消信号只停在入口，数据库查询或下游调用仍然继续。

误区二：忘记 `defer cancel()`。  
这会让定时器等资源释放不及时。

误区三：超时后继续写关键数据。  
如果请求已超时，但后台仍然提交订单或扣库存，用户重试可能造成重复操作。关键写入要有幂等设计和事务保护。

## 七、本节练习

练习一：把 `context.WithTimeout` 的时间改成 `300ms`，观察用户资料是否能加载成功。然后再改回 `100ms`，对比两次输出。

练习二：在 `loadProfile` 中增加第二步模拟加载用户订单数量，每一步都要监听 `ctx.Done()`。这个练习是为了养成习惯：长链路任务不要只在入口检查一次取消。

练习三：把示例中的数据库注释补成你熟悉的驱动写法，例如 MySQL 的 `?` 或 PostgreSQL 的 `$1`，并写一句注释说明不能拼接用户输入。这里练的是工程边界，不是 SQL 语法。

## 八、本节达标标准

- 能用 `context.WithTimeout` 控制请求超时。
- 能在 `select` 中监听 `ctx.Done()`。
- 能说明 channel 与 context 的分工。
- 能知道数据库和 HTTP 调用要使用支持 context 的方法。
