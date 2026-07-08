# 13. 综合项目：订单通知 TCP 网关

本节目标：学完后你能把监听、并发、协议、超时、日志和优雅退出组合成一个可运行的小型 TCP 网关。

简短引入：前面每节只讲一个点，这一节把它们串起来。我们实现一个订单通知 TCP 网关：客户端发送一行 JSON，服务端校验后返回 `OK` 或 `ERR`。它不是完整生产系统，但结构已经接近真实后端服务的骨架。

## 一、为什么需要它

综合项目的价值在于把零散 API 变成工程流程：

- 服务如何启动。
- 连接如何接入。
- 请求如何分隔和校验。
- 超时如何设置。
- 错误如何返回和记录。
- 服务如何退出。

```text
后端项目不是把 API 串起来，而是让正常路径和失败路径都可控。
```

## 二、基本用法

创建目录：

Windows PowerShell：

```powershell
mkdir order-tcp-gateway
cd order-tcp-gateway
go mod init order-tcp-gateway
```

Linux/macOS：

```bash
mkdir order-tcp-gateway
cd order-tcp-gateway
go mod init order-tcp-gateway
```

创建 `main.go`：

```go
package main

import (
	"bufio"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"net"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"
	"time"
)

type OrderNotice struct {
	OrderID string `json:"order_id"`
	UserID  string `json:"user_id"`
	Amount  int64  `json:"amount"`
}

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	ln, err := net.Listen("tcp", "127.0.0.1:9010")
	if err != nil {
		fmt.Println("listen failed:", err)
		return
	}

	var wg sync.WaitGroup
	sem := make(chan struct{}, 50)

	go func() {
		<-ctx.Done()
		fmt.Println("event=shutdown_signal")
		_ = ln.Close()
	}()

	fmt.Println("event=listen addr=" + ln.Addr().String())

	for {
		conn, err := ln.Accept()
		if err != nil {
			if errors.Is(err, net.ErrClosed) {
				break
			}
			fmt.Println("event=accept_error err=" + err.Error())
			continue
		}

		select {
		case sem <- struct{}{}:
			wg.Add(1)
			go func() {
				defer wg.Done()
				defer func() { <-sem }()
				handleConn(conn)
			}()
		default:
			fmt.Fprintln(conn, "ERR server_busy")
			_ = conn.Close()
		}
	}

	done := make(chan struct{})
	go func() {
		wg.Wait()
		close(done)
	}()

	select {
	case <-done:
		fmt.Println("event=shutdown_done")
	case <-time.After(5 * time.Second):
		fmt.Println("event=shutdown_timeout")
	}
}

func handleConn(conn net.Conn) {
	start := time.Now()
	remote := conn.RemoteAddr().String()
	defer conn.Close()

	_ = conn.SetDeadline(time.Now().Add(5 * time.Second))

	reader := bufio.NewReader(conn)
	line, err := reader.ReadString('\n')
	if err != nil {
		fmt.Fprintln(conn, "ERR bad_request")
		fmt.Printf("event=read_error remote=%s err=%v cost=%s\n", remote, err, time.Since(start))
		return
	}

	notice, err := parseNotice(line)
	if err != nil {
		fmt.Fprintln(conn, "ERR invalid_notice")
		fmt.Printf("event=invalid_notice remote=%s err=%v cost=%s\n", remote, err, time.Since(start))
		return
	}

	if err := saveNotice(notice); err != nil {
		fmt.Fprintln(conn, "ERR save_failed")
		fmt.Printf("event=save_error remote=%s order_id=%s err=%v cost=%s\n", remote, notice.OrderID, err, time.Since(start))
		return
	}

	fmt.Fprintln(conn, "OK")
	fmt.Printf("event=notice_ok remote=%s order_id=%s user_id=%s amount=%d cost=%s\n",
		remote, notice.OrderID, notice.UserID, notice.Amount, time.Since(start))
}

func parseNotice(line string) (OrderNotice, error) {
	line = strings.TrimSpace(line)
	if len(line) > 4096 {
		return OrderNotice{}, fmt.Errorf("notice too large")
	}

	var notice OrderNotice
	if err := json.Unmarshal([]byte(line), &notice); err != nil {
		return OrderNotice{}, err
	}

	if notice.OrderID == "" || notice.UserID == "" || notice.Amount <= 0 {
		return OrderNotice{}, fmt.Errorf("missing required fields")
	}

	return notice, nil
}

func saveNotice(notice OrderNotice) error {
	// 学习阶段先打印。真实项目中这里通常会写数据库或消息队列。
	// 写数据库时必须使用参数化查询；涉及多表写入时要明确事务边界。
	return nil
}
```

运行服务端：

```bash
go run .
```

另开终端创建客户端目录：

Windows PowerShell：

```powershell
mkdir client
```

Linux/macOS：

```bash
mkdir -p client
```

创建 `client/main.go`：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"time"
)

func main() {
	conn, err := net.DialTimeout("tcp", "127.0.0.1:9010", 2*time.Second)
	if err != nil {
		fmt.Println("dial failed:", err)
		return
	}
	defer conn.Close()

	_ = conn.SetDeadline(time.Now().Add(3 * time.Second))

	fmt.Fprintln(conn, `{"order_id":"order-1001","user_id":"user-2001","amount":9900}`)

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
go run ./client
```

## 三、关键参数/语法/代码结构

`sem := make(chan struct{}, 50)`：

- 限制同时处理的连接数。
- 超过限制时快速返回 `ERR server_busy`。

`conn.SetDeadline(time.Now().Add(5 * time.Second))`：

- 限制单连接的总处理时间。
- 防止慢连接长期占用资源。

`ReadString('\n')`：

- 使用换行作为消息边界。
- 客户端必须以换行结束一条 JSON。

`parseNotice`：

- 专门负责协议和字段校验。
- 不在网络处理函数里混入太多业务细节。

`saveNotice`：

- 代表业务落地位置。
- 真实项目中可以写数据库、发消息队列或调用内部服务。

## 四、真实后端场景示例

在真实订单系统中，这个网关可以接收外部合作方通知：

1. 合作方连接 TCP 网关。
2. 发送一行订单通知 JSON。
3. 网关校验字段和金额。
4. 网关写入数据库或消息队列。
5. 返回 `OK` 表示接收成功。

如果要写数据库，保守做法是：

```text
用户输入不能直接拼进 SQL 字符串。
```

并且要考虑：

- `order_id` 是否有唯一索引，避免重复通知重复入库。
- 写入订单通知和更新订单状态是否需要同一个事务。
- 如果写消息队列成功但写数据库失败，是否有补偿方案。
- 数据库迁移要用迁移工具管理，发布失败要能回滚。
- 索引能加速查询，但会增加写入成本，不要为了“可能会查”随意加索引。

## 五、注意点

- 示例协议没有鉴权，生产环境必须加签名、token 或 mTLS。
- 示例只读一条消息就关闭连接，适合通知类短连接。
- 如果要做长连接，需要循环读取，并在每条消息后刷新 deadline。
- JSON 最大长度要限制，避免大包打爆内存。
- 返回给客户端的错误要简洁，内部错误写日志，不要暴露堆栈。

## 六、常见误区

- 误区一：把这个示例直接当生产网关。  
  生产还需要鉴权、指标、限流、压测、部署配置、告警和回滚方案。

- 误区二：只校验 JSON 能解析。  
  业务字段也要校验，例如订单号、金额、用户 ID。

- 误区三：收到通知后直接更新多个表但不用事务。  
  中途失败会留下不一致数据，状态变更要设计事务边界和幂等键。

- 误区四：没有唯一约束。  
  外部通知可能重试，`order_id` 或通知 ID 应有唯一约束来兜底幂等。

## 七、本节达标标准

- 能运行订单通知 TCP 网关和客户端。
- 能说明监听、并发、协议、超时、日志、优雅退出在项目中的作用。
- 能把网络输入校验和业务保存逻辑拆开。
- 能列出把示例升级到生产服务前需要补齐的能力。
