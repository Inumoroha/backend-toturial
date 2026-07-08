# 1. 认识 net：后端服务的网络入口

本节目标：学完后你能说清楚 Go 的 `net` 包在后端项目中的位置，并能运行一个最小网络连接示例。

简短引入：平时写后端接口时，我们更多接触 `net/http`、Gin、gRPC。它们看起来离网络很远，但底层都离不开连接、监听端口、读写数据、处理超时。`net` 包就是 Go 标准库里处理这些基础网络动作的入口。

## 一、为什么需要它

真实项目中，你不一定每天直接写 `net.Listen`，但你一定会遇到这些问题：

- 服务端口为什么被占用。
- 客户端为什么一直连不上。
- 请求为什么偶尔超时。
- 长连接为什么慢慢把服务拖垮。
- 反向代理、网关、RPC 框架为什么要设置连接池和超时。

可以理解为，`net` 是后端网络能力的地基。你学它，不是为了以后所有东西都手写，而是为了看懂更高层框架在替你做什么。

```text
业务代码可以站在框架上，但排查问题时通常要回到连接、端口、超时和日志。
```

## 二、基本用法

下面例子会连接一个本机 TCP 服务。先不用关心服务端怎么写，只看客户端连接动作。

新建目录：

Windows PowerShell：

```powershell
mkdir net-hello
cd net-hello
go mod init net-hello
```

Linux/macOS：

```bash
mkdir net-hello
cd net-hello
go mod init net-hello
```

创建 `main.go`：

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
		fmt.Println("connect failed:", err)
		return
	}
	defer conn.Close()

	fmt.Println("connected:", conn.RemoteAddr())
}
```

运行：

```bash
go run .
```

这个程序做了三件事：连接远端地址、失败时返回错误、成功后关闭连接。

## 三、关键参数/语法/代码结构

`net.DialTimeout("tcp", "example.com:80", 3*time.Second)`：

- `"tcp"` 表示使用 TCP 协议。后端服务常见的 HTTP、MySQL、Redis、gRPC 都建立在 TCP 之上。
- `"example.com:80"` 是地址，格式通常是 `host:port`。
- `3*time.Second` 是连接超时。真实项目中不能无限等，否则一个下游故障会拖住调用方。

`defer conn.Close()`：

- 连接是系统资源，使用完必须关闭。
- 在服务端长连接场景中，关闭时机要更谨慎，但“谁创建、谁负责释放”的原则不变。

## 四、真实后端场景示例

假设你的订单服务启动时要检查库存服务是否能连通，可以用类似方式做启动前探测：

```go
package main

import (
	"fmt"
	"net"
	"time"
)

func main() {
	addr := "inventory.internal:9000"

	conn, err := net.DialTimeout("tcp", addr, 2*time.Second)
	if err != nil {
		fmt.Println("inventory service unavailable:", err)
		return
	}
	defer conn.Close()

	fmt.Println("inventory service reachable:", conn.RemoteAddr())
}
```

真实项目中通常不会只靠启动探测保证服务可用，因为服务可能启动后才故障。更常见的做法是：每次调用设置超时、记录错误、必要时走降级逻辑。

## 五、注意点

- `net` 只负责网络 I/O，不负责业务协议。你写入什么、如何分隔消息、如何校验数据，需要自己设计。
- 直接使用 `net` 更灵活，也更容易犯错。Web API 优先使用 `net/http` 或成熟框架。
- 不要在生产服务里无限等待连接或读写。
- 网络输入都不可信，后续写入数据库时仍要使用参数化查询，并把事务边界放在业务层清楚控制。

```text
网络连通不等于业务可用；一次探测成功也不等于后续请求一定成功。
```

## 六、常见误区

- 误区一：学后端只学 HTTP 就够了。  
  HTTP 是应用层协议，排查连接问题、超时问题、代理问题时仍然需要网络基础。

- 误区二：连接失败就是对方服务挂了。  
  也可能是 DNS、端口、防火墙、路由、超时配置或本机网络问题。

- 误区三：示例能跑就可以上线。  
  示例通常缺少重试、限流、日志、指标、超时和关闭流程，只能作为学习起点。

## 七、本节达标标准

- 能解释 `net` 包和 `net/http` 的关系。
- 能运行一个 `net.DialTimeout` 示例。
- 能说出网络代码里为什么必须设置超时。
- 能说明连接用完后为什么要关闭。
