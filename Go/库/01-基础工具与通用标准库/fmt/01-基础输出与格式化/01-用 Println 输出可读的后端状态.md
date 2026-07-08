# 01. 用 Println 输出可读的后端状态

本节目标：学会用 `fmt.Print`、`fmt.Println` 输出简单业务状态，并知道它们适合放在哪里。

简短引入：刚开始写 Go 后端时，你会经常想确认“服务启动了吗”“这条订单处理到哪了”“这个用户 ID 是多少”。这时 `fmt.Println` 是最直接的工具。

## 一、为什么需要它

可以把 `Println` 理解为最轻量的观察窗口。它适合学习、临时脚本、本地调试，不适合作为生产日志体系。

真实项目中常见场景是：

- 本地运行 HTTP 服务时打印监听地址。
- 写一次性数据修复脚本时打印处理进度。
- 学习新包时临时确认变量内容。

```text
生产服务不要长期依赖 fmt.Println 做日志，真实项目通常会使用 log/slog 或成熟日志库。
```

## 二、基本用法

新建 `main.go`：

```go
package main

import "fmt"

func main() {
	userID := 1001
	orderID := "ORD-20260706-0001"

	fmt.Print("start processing ")
	fmt.Println(orderID)
	fmt.Println("user id:", userID)
}
```

Windows PowerShell：

```powershell
go run .\main.go
```

Linux/macOS：

```bash
go run ./main.go
```

## 三、关键参数/语法/代码结构

`fmt.Print` 会按默认格式连续输出，不会自动换行。

`fmt.Println` 会在参数之间自动加空格，并在末尾加换行。后端脚本里打印一条处理结果时，通常优先用它，因为输出一行就是一个事件。

`fmt.Println("user id:", userID)` 比字符串拼接更省心，因为整数不需要手动转换成字符串。

## 四、真实后端场景示例

下面模拟一个批量关闭过期订单的脚本：

```go
package main

import "fmt"

func main() {
	expiredOrders := []string{"ORD-001", "ORD-002", "ORD-003"}

	fmt.Println("start close expired orders, count:", len(expiredOrders))

	for _, orderID := range expiredOrders {
		fmt.Println("closed order:", orderID)
	}

	fmt.Println("job finished")
}
```

这个例子中，每行输出都代表脚本执行的一个阶段。以后换成 `slog.Info` 时，思路也是类似的：让输出能帮助你定位进度和问题。

## 五、注意点

`Println` 适合输出给开发者看的文本，不适合输出给用户看的正式响应。用户响应通常要考虑格式、状态码、国际化和错误码。

如果输出的是错误，真实项目中通常应写到日志或返回给调用方，而不是只打印一下就继续执行。

判断是否还能继续用 `Println`，可以看三个问题：

- 这段输出只是本地学习或一次性脚本吗？如果是，可以用。
- 这段输出需要按用户、订单、请求 ID 检索吗？如果需要，应换成结构化日志。
- 这段输出失败后会影响数据一致性吗？如果会，应返回错误并设计回滚或补偿。

例如批量修复订单状态时，`Println` 可以提示当前处理到哪一条，但修复失败不能只打印后跳过。真实项目中通常会记录失败订单、失败原因，并让脚本可以从某个批次继续执行。

## 六、常见误区

把 `fmt.Println` 当成错误处理。打印错误不等于处理错误，错误仍然需要返回、重试、回滚或终止流程。

在高并发服务里大量使用 `Println`。标准输出可能变成性能和排查问题的瓶颈，也不方便做日志检索。

输出敏感信息。用户手机号、Token、密码、身份证号不要直接打印。

## 七、本节达标标准

- 能区分 `Print` 和 `Println` 的换行行为。
- 能用 `Println` 打印脚本进度和简单业务字段。
- 知道生产日志不应长期依赖 `fmt.Println`。
- 知道敏感字段不能直接输出。
