# 4. net.Listen：写一个最小 TCP 服务

本节目标：学完后你能写出一个可以监听端口、接收连接并返回响应的 TCP 服务。

简短引入：服务端和客户端最大的不同是：服务端要长期监听端口，持续接收连接。HTTP 服务、RPC 服务、数据库服务，本质上都需要先完成“监听端口并接入连接”这件事。

## 一、为什么需要它

真实后端服务启动后，会绑定一个端口等待请求。服务启动失败，最常见的原因之一就是端口不可用，例如：

- 端口已经被另一个进程占用。
- 没有权限监听低位端口。
- 容器里监听地址和端口映射不一致。
- 服务监听了 `127.0.0.1`，但你希望外部访问。

学习 `net.Listen` 可以帮助你理解这些问题从哪里来。

## 二、基本用法

创建 `server.go`：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"strings"
)

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:9000")
	if err != nil {
		fmt.Println("listen failed:", err)
		return
	}
	defer ln.Close()

	fmt.Println("server listening on", ln.Addr())

	for {
		conn, err := ln.Accept()
		if err != nil {
			fmt.Println("accept failed:", err)
			continue
		}

		handle(conn)
	}
}

func handle(conn net.Conn) {
	defer conn.Close()

	reader := bufio.NewReader(conn)
	line, err := reader.ReadString('\n')
	if err != nil {
		fmt.Fprintln(conn, "ERR read failed")
		return
	}

	name := strings.TrimSpace(line)
	if name == "" {
		fmt.Fprintln(conn, "ERR empty name")
		return
	}

	fmt.Fprintf(conn, "OK hello %s\n", name)
}
```

运行服务端：

```bash
go run server.go
```

再开一个终端，创建 `client.go`：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"time"
)

func main() {
	conn, err := net.DialTimeout("tcp", "127.0.0.1:9000", 2*time.Second)
	if err != nil {
		fmt.Println("dial failed:", err)
		return
	}
	defer conn.Close()

	fmt.Fprintln(conn, "alice")

	reply, err := bufio.NewReader(conn).ReadString('\n')
	if err != nil {
		fmt.Println("read failed:", err)
		return
	}
	fmt.Print(reply)
}
```

运行客户端：

```bash
go run client.go
```

## 三、关键参数/语法/代码结构

`net.Listen("tcp", "127.0.0.1:9000")`：

- 创建 TCP 监听器。
- `127.0.0.1` 表示只允许本机访问。
- `9000` 是服务端口。

`ln.Accept()`：

- 等待下一个客户端连接。
- 没有连接时会阻塞。
- 返回的 `conn` 表示这一个客户端连接。

`handle(conn)`：

- 处理当前连接。
- 本节示例先串行处理，下一节会改成并发。

## 四、真实后端场景示例

可以把这个服务理解为一个非常简化的用户问候服务：

- 客户端发送用户名。
- 服务端校验用户名不能为空。
- 服务端返回业务响应。

真实后端里，这一步之后可能会查数据库、读缓存、写日志。此时要记住：

```text
网络层负责接入请求；业务层负责校验、事务、参数化查询和可回滚的状态变更。
```

如果你把用户输入直接拼进 SQL，哪怕网络服务写得再好，也会留下安全问题。

## 五、注意点

- `Accept` 失败不一定要立刻退出，有些错误可以记录后继续。
- 但如果监听器已经关闭，继续循环打印错误会刷爆日志，后面优雅退出章节会处理。
- 串行 `handle(conn)` 只能一次处理一个客户端，不适合真实服务。
- 监听地址要根据部署环境选择，开发环境不要误暴露管理端口。

## 六、常见误区

- 误区一：服务端只要 `Listen` 成功就算稳定。  
  还要考虑并发、超时、连接数、日志和退出流程。

- 误区二：在 `Accept` 循环里遇到任何错误都直接 `return`。  
  有些临时错误可以继续，但关闭监听器时要识别并退出。

- 误区三：把所有逻辑写在 `handle` 里。  
  示例可以这样写，真实项目应拆分协议解析、业务处理、存储访问和日志。

## 七、本节达标标准

- 能写出一个最小 TCP 服务端。
- 能解释 `Listen`、`Accept`、`Conn` 的关系。
- 能用客户端连接服务端并得到响应。
- 能说明串行处理连接为什么不适合真实后端服务。
