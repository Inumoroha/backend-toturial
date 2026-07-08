# 9. UDP：适合日志、探测和低成本通知的场景

本节目标：学完后你能写一个最小 UDP 接收端和发送端，并判断什么时候不该使用 UDP。

简短引入：UDP 不需要建立连接，发送成本低，延迟小，但不保证送达、不保证顺序、不保证不重复。后端项目中它常见于日志采集、指标上报、服务探测、局域网通知等场景。

## 一、为什么需要它

如果你要上报访问日志或指标，偶尔丢一条通常可以接受；如果你要扣库存、创建订单、支付回调，就绝对不能靠 UDP 的“尽力而为”。

```text
UDP 适合可丢、可重复、可乱序的消息；核心交易链路不要用它承载状态变更。
```

## 二、基本用法

UDP 接收端：

```go
package main

import (
	"fmt"
	"net"
)

func main() {
	conn, err := net.ListenPacket("udp", "127.0.0.1:9004")
	if err != nil {
		fmt.Println("listen udp failed:", err)
		return
	}
	defer conn.Close()

	fmt.Println("udp server listening on", conn.LocalAddr())

	buf := make([]byte, 1024)
	for {
		n, addr, err := conn.ReadFrom(buf)
		if err != nil {
			fmt.Println("read failed:", err)
			continue
		}
		fmt.Printf("from=%s msg=%s\n", addr.String(), string(buf[:n]))
	}
}
```

UDP 发送端：

```go
package main

import (
	"fmt"
	"net"
)

func main() {
	conn, err := net.Dial("udp", "127.0.0.1:9004")
	if err != nil {
		fmt.Println("dial udp failed:", err)
		return
	}
	defer conn.Close()

	_, err = fmt.Fprintln(conn, `access_log user=1001 path=/articles/1 status=200`)
	if err != nil {
		fmt.Println("send failed:", err)
		return
	}

	fmt.Println("sent")
}
```

先运行接收端，再运行发送端：

```bash
go run server.go
go run client.go
```

## 三、关键参数/语法/代码结构

`net.ListenPacket("udp", addr)`：

- 创建 UDP 监听。
- 返回 `PacketConn`，适合按数据包读取。

`ReadFrom(buf)`：

- 读取一个 UDP 数据报。
- 返回发送方地址。

`net.Dial("udp", addr)`：

- UDP 的 `Dial` 不代表建立可靠连接。
- 可以理解为设置默认发送目标，后续用 `Write` 发给它。

## 四、真实后端场景示例

访问日志采集可以使用 UDP：

- Web 服务每次请求后发送一条日志。
- 日志服务接收后批量写入队列或文件。
- 如果 UDP 偶尔丢包，不影响主业务响应。

但如果日志有审计要求，例如登录审计、资金操作审计，就不能只靠 UDP。应使用可靠队列、数据库或事务日志，并设计补偿与回滚流程。

再看一个服务探测场景：网关定时向内部节点发送 `PING service=order`，节点收到后返回 `PONG service=order status=ok`。这种探测允许偶尔丢包，因为下一轮还会继续探测。它的价值是成本低、实现简单、对被探测服务压力小。

如果你希望发送端等待响应，可以用 `ReadFrom` 读取返回包，但仍要设置 deadline：

```go
package main

import (
	"fmt"
	"net"
	"time"
)

func main() {
	conn, err := net.ListenPacket("udp", "127.0.0.1:0")
	if err != nil {
		fmt.Println("listen failed:", err)
		return
	}
	defer conn.Close()

	target, err := net.ResolveUDPAddr("udp", "127.0.0.1:9004")
	if err != nil {
		fmt.Println("resolve failed:", err)
		return
	}

	_ = conn.SetDeadline(time.Now().Add(1 * time.Second))

	_, err = conn.WriteTo([]byte("PING service=order"), target)
	if err != nil {
		fmt.Println("send failed:", err)
		return
	}

	buf := make([]byte, 1024)
	n, addr, err := conn.ReadFrom(buf)
	if err != nil {
		fmt.Println("probe timeout or failed:", err)
		return
	}

	fmt.Printf("from=%s reply=%s\n", addr.String(), string(buf[:n]))
}
```

真实项目里，UDP 探测结果通常不能直接决定“服务下线”。更稳妥的做法是连续多次失败才摘除节点，并保留人工回滚入口。

## 五、注意点

- UDP 包大小要控制，过大可能被分片，分片丢一个整个包就不可用。
- UDP 没有连接级背压，接收端慢时发送端不一定知道。
- UDP 数据也来自不可信网络，仍要校验格式、来源和大小。
- 生产环境要考虑防火墙、安全组和服务暴露范围。
- UDP 服务端要限制单包处理耗时，不要在读取循环里做很重的数据库操作。
- 如果消息不能丢，优先考虑 TCP、HTTP、可靠消息队列或数据库事务表。
- 如果 UDP 用于指标上报，发送失败通常只记录采样日志，不要阻塞主业务请求。

## 六、常见误区

- 误区一：UDP 更快，所以所有场景都该用。  
  快的代价是不可靠，不适合订单、支付、库存等核心状态变更。

- 误区二：以为 `Dial("udp")` 建立了真实连接。  
  它主要是绑定默认目标，并不保证对方在线。

- 误区三：日志用 UDP 就一定安全。  
  普通访问日志可以接受丢失，审计日志通常不能接受。

- 误区四：收到一次探测失败就立刻下线服务。  
  UDP 本身可能丢包，真实项目通常要连续失败、结合其他健康信号再决策。

## 七、本节达标标准

- 能写出 UDP 接收端和发送端。
- 能解释 UDP 不保证送达、顺序和唯一性。
- 能判断日志、指标、探测等场景是否适合 UDP。
- 能说明为什么核心状态变更不要直接依赖 UDP。
- 能为 UDP 探测设置 deadline，并知道失败结果不能直接等同于服务不可用。
