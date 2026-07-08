# 2. 地址、端口与 net.Addr

本节目标：学完后你能正确处理 `host:port`，避免在后端配置、IPv6、端口解析中踩坑。

简短引入：后端服务最终都要绑定或访问某个地址。地址看起来只是字符串，但真实项目里常常来自配置中心、环境变量、命令行参数或服务发现结果。如果随手拼字符串，后面遇到 IPv6、空 host、非法端口时会很难排查。

## 一、为什么需要它

常见业务场景：

- 用户服务监听 `:8080`，供网关转发。
- 订单服务访问库存服务 `inventory:9000`。
- 管理后台只允许监听 `127.0.0.1:7001`。
- 测试环境端口从环境变量读取。

地址处理的目标不是炫技，而是减少隐蔽错误。

```text
真实项目里，地址通常来自配置；凡是来自配置，就要解析、校验、记录。
```

## 二、基本用法

创建 `main.go`：

```go
package main

import (
	"fmt"
	"net"
)

func main() {
	addr := net.JoinHostPort("127.0.0.1", "8080")
	fmt.Println("joined:", addr)

	host, port, err := net.SplitHostPort(addr)
	if err != nil {
		fmt.Println("split failed:", err)
		return
	}

	fmt.Println("host:", host)
	fmt.Println("port:", port)

	tcpAddr, err := net.ResolveTCPAddr("tcp", addr)
	if err != nil {
		fmt.Println("resolve failed:", err)
		return
	}

	fmt.Println("resolved:", tcpAddr.Network(), tcpAddr.String())
}
```

运行：

```bash
go run .
```

## 三、关键参数/语法/代码结构

`net.JoinHostPort(host, port)`：

- 用来安全拼接地址。
- IPv6 地址里包含冒号，手写 `host + ":" + port` 容易出错。

`net.SplitHostPort(addr)`：

- 用来拆分地址。
- 如果地址格式不合法，会返回错误。

`net.ResolveTCPAddr("tcp", addr)`：

- 把字符串解析成 `*net.TCPAddr`。
- 常见于你需要提前校验配置，或使用更具体的 TCP API。

## 四、真实后端场景示例

假设服务启动时从环境变量读取监听地址：

```go
package main

import (
	"fmt"
	"net"
	"os"
)

func main() {
	host := os.Getenv("APP_HOST")
	if host == "" {
		host = "127.0.0.1"
	}

	port := os.Getenv("APP_PORT")
	if port == "" {
		port = "8080"
	}

	addr := net.JoinHostPort(host, port)
	if _, err := net.ResolveTCPAddr("tcp", addr); err != nil {
		fmt.Println("invalid listen address:", err)
		return
	}

	fmt.Println("service will listen on", addr)
}
```

Windows PowerShell：

```powershell
$env:APP_HOST="127.0.0.1"
$env:APP_PORT="8080"
go run .
```

Linux/macOS：

```bash
APP_HOST=127.0.0.1 APP_PORT=8080 go run .
```

真实项目中通常会在启动阶段校验配置，尽早失败，而不是等到处理请求时才发现地址错误。

## 五、注意点

- `:8080` 表示监听本机所有可用网卡，开发方便，但生产环境要确认是否符合安全策略。
- `127.0.0.1:8080` 只允许本机访问，适合管理端口、本机代理或 sidecar。
- `0.0.0.0:8080` 常用于容器内监听，但对外暴露由容器端口映射和安全组决定。
- 不要把端口写死在多个地方，应该通过配置统一管理。

## 六、常见误区

- 误区一：直接用字符串拼地址。  
  IPv4 看起来没问题，遇到 IPv6 或空 host 时就容易出错。

- 误区二：以为监听 `:8080` 一定只能本机访问。  
  它通常表示监听所有网卡，外部能不能访问取决于网络环境、防火墙和部署方式。

- 误区三：配置读取后不校验。  
  服务启动成功但实际无法监听，会把故障推迟到更难排查的阶段。

## 七、本节达标标准

- 能使用 `JoinHostPort` 拼接地址。
- 能使用 `SplitHostPort` 拆分地址并处理错误。
- 能说明 `127.0.0.1:8080`、`:8080`、`0.0.0.0:8080` 的区别。
- 能在服务启动时校验监听地址配置。
