# 01. 认识日志与标准库 log

本节目标：学完后，你能在 Go 程序中使用标准库 `log` 输出有时间信息的基础日志。

## 简短引入

后端服务运行起来以后，很多问题不是当场暴露给开发者看的。用户创建订单失败、文章发布超时、库存扣减异常，这些都需要日志帮助我们还原现场。Go 标准库的 `log` 很简单，适合初学阶段建立日志直觉，也适合小工具、脚本、迁移程序使用。

## 一、为什么需要它

可以理解为，日志是程序在运行过程中留下的“事实记录”。真实项目中常见场景是：

- 用户注册失败，需要知道是参数错误、数据库错误还是验证码错误。
- 定时任务执行失败，需要知道处理到哪一条数据。
- 线上服务偶发 500，需要知道当时的请求路径、用户、错误原因。

只用 `fmt.Println` 也能打印内容，但它默认没有时间、没有统一格式，也不适合长期维护。

```text
日志记录事实，错误处理决定程序接下来怎么走；两者不能互相替代。
```

## 二、基本用法

新建一个临时目录运行下面代码。

Windows PowerShell：

```powershell
mkdir go-log-demo
cd go-log-demo
go mod init go-log-demo
New-Item main.go
```

Linux/macOS：

```bash
mkdir go-log-demo
cd go-log-demo
go mod init go-log-demo
touch main.go
```

把下面代码复制到 `main.go`：

```go
package main

import (
	"log"
)

func main() {
	log.Println("user register started")
	log.Println("user register success", "user_id", 1001)
}
```

运行：

```bash
go run .
```

你会看到类似输出：

```text
2026/07/06 10:20:30 user register started
2026/07/06 10:20:30 user register success user_id 1001
```

## 三、关键参数/语法/代码结构

`log.Println` 会自动输出日期和时间。它和 `fmt.Println` 很像，但面向的是运行日志，不是普通控制台提示。

标准库 `log` 常见函数：

- `log.Print`：输出日志，不额外换行。
- `log.Println`：输出日志，并换行，初学阶段最常用。
- `log.Printf`：按格式输出，适合拼接少量变量。
- `log.Fatal`：输出日志后调用 `os.Exit(1)` 退出程序。
- `log.Panic`：输出日志后触发 panic。

真实项目中，`Fatal` 和 `Panic` 要非常谨慎。比如 HTTP 请求处理函数里不应该随便 `log.Fatal`，因为它会让整个进程退出。

## 四、真实后端场景示例

下面用“创建文章”的场景演示基础日志。

```go
package main

import (
	"errors"
	"log"
)

func createArticle(userID int64, title string) error {
	log.Println("create article started", "user_id", userID, "title", title)

	if title == "" {
		return errors.New("title is empty")
	}

	log.Println("create article success", "user_id", userID)
	return nil
}

func main() {
	if err := createArticle(1001, "Go logging guide"); err != nil {
		log.Println("create article failed", "error", err)
	}
}
```

这个例子里，函数内部记录业务开始和成功，调用方记录失败。真实项目中通常还会加上文章 ID、请求 ID、耗时等信息。

## 五、注意点

日志内容要围绕排查问题。不要只写 `failed`，而要写清楚哪个动作失败、关键参数是什么、错误是什么。

不要把所有变量都打印出来。比如用户密码、Token、验证码、身份证号不能直接写入日志。

小工具或学习代码可以直接用标准库 `log`。中大型 Web 服务通常会进一步使用 `log/slog` 或第三方日志库，方便输出 JSON 和接入日志平台。

## 六、常见误区

误区一：用 `fmt.Println` 代替日志。  
这样做短期方便，但后续很难统一格式，也不利于排查。

误区二：遇到错误就 `log.Fatal`。  
`Fatal` 会退出进程，适合程序启动阶段遇到无法继续运行的问题，比如配置文件不存在。普通请求失败不应该退出整个服务。

误区三：只记录“失败了”。  
没有动作、参数、错误原因的日志，排查价值很低。

## 七、本节达标标准

- 能使用 `log.Println` 输出基础日志。
- 能说清楚日志和普通打印的区别。
- 知道 `Fatal` 不适合在普通业务请求中随便使用。
- 能在一个简单业务函数中记录开始、成功和失败。

