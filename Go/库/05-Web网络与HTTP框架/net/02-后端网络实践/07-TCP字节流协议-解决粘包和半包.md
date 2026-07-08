# 7. TCP 字节流协议：解决粘包和半包

本节目标：学完后你能在 TCP 字节流上设计简单消息边界，避免把一次读写误认为一条业务消息。

简短引入：TCP 传的是连续字节流，不是“消息列表”。你写两次，对方可能一次读到；你写一次，对方也可能分两次读到。这就是初学者常说的粘包和半包。

## 一、为什么需要它

后端业务通常处理的是一条条命令：

- `CREATE_USER {"name":"alice"}`
- `DEDUCT_STOCK {"sku":"book","count":1}`
- `CREATE_SHORTLINK {"url":"https://example.com"}`

但 TCP 不知道哪里是一条命令的结束。你必须自己约定消息边界。

```text
TCP 只负责把字节按顺序送到；业务消息从哪里开始、到哪里结束，要由协议定义。
```

## 二、基本用法

最简单的做法是换行协议：一行就是一条消息。

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"strings"
)

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:9003")
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
	defer conn.Close()

	scanner := bufio.NewScanner(conn)
	for scanner.Scan() {
		msg := strings.TrimSpace(scanner.Text())
		if msg == "" {
			fmt.Fprintln(conn, "ERR empty command")
			continue
		}
		fmt.Fprintf(conn, "OK received %q\n", msg)
	}

	if err := scanner.Err(); err != nil {
		fmt.Println("scan failed:", err)
	}
}
```

客户端：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"time"
)

func main() {
	conn, err := net.DialTimeout("tcp", "127.0.0.1:9003", 2*time.Second)
	if err != nil {
		fmt.Println("dial failed:", err)
		return
	}
	defer conn.Close()

	fmt.Fprintln(conn, `CREATE_SHORTLINK {"url":"https://example.com/a"}`)
	fmt.Fprintln(conn, `CREATE_SHORTLINK {"url":"https://example.com/b"}`)

	reader := bufio.NewReader(conn)
	for i := 0; i < 2; i++ {
		reply, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("read failed:", err)
			return
		}
		fmt.Print(reply)
	}
}
```

## 三、关键参数/语法/代码结构

`bufio.Scanner`：

- 默认按行扫描。
- 适合小消息、文本命令、学习示例。
- 默认单个 token 有大小限制，大消息不适合直接用默认配置。

`fmt.Fprintln(conn, msg)`：

- 写入内容并追加换行。
- 对换行协议来说，换行就是消息结束标记。

长度前缀协议：

- 另一种常见做法是在消息前写入长度。
- 更适合二进制协议或消息体里可能包含换行的场景。
- 学习初期先掌握换行协议，后续再扩展长度前缀。

## 四、真实后端场景示例

短链接内部服务可以约定：

```text
CREATE_SHORTLINK {"url":"https://example.com/articles/1001"}
```

服务端按行读取后：

1. 解析命令名。
2. 校验 JSON。
3. 调用业务层创建短码。
4. 返回 `OK abc123` 或 `ERR reason`。

真实项目中，如果命令会落库：

- URL 必须校验长度和格式。
- SQL 必须参数化。
- 创建短码和写入映射应在清楚的事务边界内。
- 失败时要能回滚或安全重试。

## 五、注意点

- 换行协议要求消息体不能随意包含未转义换行。
- Scanner 默认限制单行大小，大 payload 要改用 `bufio.Reader` 并限制最大长度。
- 不要无限读取超大消息，否则可能被恶意请求打爆内存。
- 协议错误要返回明确响应，但不要把内部堆栈暴露给客户端。

## 六、常见误区

- 误区一：认为 `Read` 一次就是一条消息。  
  这是 TCP 初学最常见错误，会导致偶发解析失败。

- 误区二：没有最大消息长度。  
  客户端可以发送超大内容拖垮内存。

- 误区三：协议解析和业务落库混在一起。  
  这样很难测试，也很难保证事务和错误处理清晰。

## 七、本节达标标准

- 能解释 TCP 为什么会出现粘包和半包。
- 能用换行协议处理多条业务消息。
- 能说出换行协议的适用边界。
- 能说明为什么网络输入要限制长度并校验。
