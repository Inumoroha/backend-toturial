# 3. net.Conn：一次 TCP 连接的生命周期

本节目标：学完后你能理解一次 TCP 连接从创建、读写到关闭的基本流程。

简短引入：`net.Conn` 可以理解为一条已经建立好的网络通道。你可以从里面读数据，也可以往里面写数据。后端里的数据库连接、Redis 连接、HTTP keep-alive 连接，本质上都绕不开类似的生命周期管理。

## 一、为什么需要它

真实项目中，连接不是“用完就消失”的普通变量。它背后占着文件描述符、内存、内核缓冲区等资源。连接不关闭、读写不设限制、错误不处理，服务就会在流量上来后变得不稳定。

```text
连接是资源，不是普通字符串；资源就必须有生命周期。
```

## 二、基本用法

下面写一个最小客户端，连接后发送一行文本：

```go
package main

import (
	"fmt"
	"net"
	"time"
)

func main() {
	conn, err := net.DialTimeout("tcp", "example.com:80", 3*time.Second)
	if err != nil {
		fmt.Println("dial failed:", err)
		return
	}
	defer conn.Close()

	_, err = conn.Write([]byte("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n"))
	if err != nil {
		fmt.Println("write failed:", err)
		return
	}

	buf := make([]byte, 128)
	n, err := conn.Read(buf)
	if err != nil {
		fmt.Println("read failed:", err)
		return
	}

	fmt.Println(string(buf[:n]))
}
```

运行：

```bash
go run .
```

## 三、关键参数/语法/代码结构

`conn.Read(buf)`：

- 从连接读取数据到 `buf`。
- 返回实际读取的字节数 `n`。
- 一次 `Read` 不保证读完整个业务消息。

`conn.Write(data)`：

- 向连接写入字节。
- 返回写入字节数和错误。
- 真实项目中要检查错误，尤其是写超时、连接断开。

`conn.Close()`：

- 关闭连接。
- 多数情况下客户端用完就关；服务端要在处理完成、出错或退出时关。

## 四、真实后端场景示例

假设你要调用一个内部短链接服务，它使用简单 TCP 协议：发送 `CREATE https://example.com\n`，服务返回短码。

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
		fmt.Println("shortlink service unavailable:", err)
		return
	}
	defer conn.Close()

	_ = conn.SetDeadline(time.Now().Add(3 * time.Second))

	if _, err := fmt.Fprintln(conn, "CREATE https://example.com/articles/1001"); err != nil {
		fmt.Println("send command failed:", err)
		return
	}

	reply, err := bufio.NewReader(conn).ReadString('\n')
	if err != nil {
		fmt.Println("read reply failed:", err)
		return
	}

	fmt.Println("short code:", reply)
}
```

这里的重点不是短链接算法，而是一次调用要有连接超时、整体读写超时、错误处理和关闭。

## 五、注意点

- TCP 是字节流，不保留消息边界。你写一次，不代表对方读一次就能完整拿到。
- `Read` 返回 `n > 0` 且 `err != nil` 时，仍要先处理已经读到的数据，再处理错误。
- `SetDeadline` 是给连接设置读写截止时间，不是业务重试。
- 连接关闭不一定意味着业务失败，可能只是对方正常处理完后断开。

## 六、常见误区

- 误区一：认为一次 `Write` 对应一次 `Read`。  
  TCP 只保证字节顺序，不保证消息边界。

- 误区二：只在出错时关闭连接。  
  成功路径也要关闭，否则会慢慢耗尽资源。

- 误区三：忽略 `Write` 的错误。  
  对方断开、网络抖动、缓冲区问题都可能让写失败。

## 七、本节达标标准

- 能说明 `net.Conn` 的读、写、关流程。
- 能写出一个带连接超时的 TCP 客户端。
- 能解释为什么 TCP 需要自己设计消息边界。
- 能在示例里正确处理 `Read`、`Write`、`Close` 的错误。
