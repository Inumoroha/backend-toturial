# 5. 并发处理连接：每个连接一个 goroutine

本节目标：学完后你能把串行 TCP 服务改造成可以同时处理多个客户端的并发服务。

简短引入：后端服务不可能一次只服务一个用户。上一节的服务在处理一个连接时，其他连接只能等着。真实项目中通常会让每个连接进入独立 goroutine，由 Go 运行时调度。

## 一、为什么需要它

假设订单通知服务每次处理连接要查库存、写日志、返回结果。如果第一个客户端处理 2 秒，串行服务会让后面的客户端都排队。并发处理可以让多个连接同时推进。

```text
并发不是为了让单个请求更快，而是为了让服务能同时处理多个请求。
```

## 二、基本用法

创建 `server.go`：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"strings"
	"sync/atomic"
)

var active int64

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:9001")
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

		go handle(conn)
	}
}

func handle(conn net.Conn) {
	atomic.AddInt64(&active, 1)
	defer atomic.AddInt64(&active, -1)
	defer conn.Close()

	reader := bufio.NewReader(conn)
	line, err := reader.ReadString('\n')
	if err != nil {
		fmt.Fprintln(conn, "ERR read failed")
		return
	}

	userID := strings.TrimSpace(line)
	if userID == "" {
		fmt.Fprintln(conn, "ERR empty user_id")
		return
	}

	fmt.Fprintf(conn, "OK user=%s active=%d\n", userID, atomic.LoadInt64(&active))
}
```

运行：

```bash
go run server.go
```

另开终端，连续运行客户端：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"time"
)

func main() {
	userID := "1001"
	if len(os.Args) > 1 {
		userID = os.Args[1]
	}

	conn, err := net.DialTimeout("tcp", "127.0.0.1:9001", 2*time.Second)
	if err != nil {
		fmt.Println("dial failed:", err)
		return
	}
	defer conn.Close()

	fmt.Fprintln(conn, userID)
	reply, err := bufio.NewReader(conn).ReadString('\n')
	if err != nil {
		fmt.Println("read failed:", err)
		return
	}
	fmt.Print(reply)
}
```

```bash
go run client.go 1001
go run client.go 1002
```

## 三、关键参数/语法/代码结构

`go handle(conn)`：

- 每个连接交给一个 goroutine。
- `Accept` 循环可以继续接收新连接。
- 这是 Go 网络服务非常常见的结构。

`defer conn.Close()`：

- 每个处理函数负责关闭自己的连接。
- 如果忘记关闭，连接数会持续增长。

`atomic.AddInt64`：

- 示例用它记录当前连接数。
- 真实项目中可以接入 Prometheus、日志或内部监控。

## 四、真实后端场景示例

假设权限服务通过 TCP 接收内部系统的权限检查：

- 客户端发送 `user_id`。
- 服务端校验格式。
- 服务端返回是否允许。

真实项目中，权限检查可能要读缓存或数据库。此时要注意：

- 数据库访问要设置上下文超时。
- SQL 必须参数化。
- 如果涉及写操作，事务边界应放在明确的业务函数里。
- 不要在每个连接里无限制创建下游连接。

## 五、注意点

- 每连接一个 goroutine 是常见做法，但不是无限制做法。
- 如果外部大量连接涌入，goroutine、内存和文件描述符都会被消耗。
- 下一步通常要增加连接数限制、读写超时和限流。
- 不要在 goroutine 里吞掉错误，至少要记录关键信息。

## 六、常见误区

- 误区一：goroutine 很轻量，所以可以无限开。  
  它比线程轻，但不是没有成本。连接数必须能观测、能限制。

- 误区二：只在主 goroutine 里处理错误。  
  连接处理错误发生在子 goroutine，也要记录。

- 误区三：共享变量随便读写。  
  多 goroutine 访问共享状态要用锁、atomic 或 channel。

## 七、本节达标标准

- 能把 `handle(conn)` 改成 `go handle(conn)`。
- 能说明并发处理连接的好处和风险。
- 能给每个连接正确关闭资源。
- 能说出为什么真实服务还需要连接数限制。
