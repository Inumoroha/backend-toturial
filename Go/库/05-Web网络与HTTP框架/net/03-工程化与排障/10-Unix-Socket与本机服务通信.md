# 10. Unix Socket 与本机服务通信

本节目标：学完后你能理解 Unix Socket 的用途，并在支持的平台上写一个本机通信示例。

简短引入：有些后端组件只需要在同一台机器或同一个 Pod 内通信，例如主服务和 sidecar、管理工具和本机代理。此时 Unix Socket 可以避免暴露 TCP 端口，权限控制也更贴近文件系统。

## 一、为什么需要它

常见场景：

- Nginx 通过 Unix Socket 转发到本机应用。
- 主进程和本机运维 agent 通信。
- 容器内服务和 sidecar 交换状态。
- 本机管理命令访问守护进程。

可以理解为，Unix Socket 是“像网络连接一样读写，但地址是一个本机文件路径”。

```text
Unix Socket 适合本机通信；跨机器通信仍然使用 TCP。
```

## 二、基本用法

注意：Unix Socket 主要用于 Linux/macOS。Windows 对 Unix Socket 的支持存在差异，学习时建议优先在 Linux/macOS 或 WSL 中运行。

服务端：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"strings"
)

func main() {
	socketPath := "/tmp/order-agent.sock"
	_ = os.Remove(socketPath)

	ln, err := net.Listen("unix", socketPath)
	if err != nil {
		fmt.Println("listen unix failed:", err)
		return
	}
	defer ln.Close()
	defer os.Remove(socketPath)

	fmt.Println("unix socket listening on", socketPath)

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

	line, err := bufio.NewReader(conn).ReadString('\n')
	if err != nil {
		fmt.Fprintln(conn, "ERR read failed")
		return
	}

	cmd := strings.TrimSpace(line)
	if cmd == "PING" {
		fmt.Fprintln(conn, "PONG")
		return
	}
	fmt.Fprintln(conn, "ERR unknown command")
}
```

客户端：

```go
package main

import (
	"bufio"
	"fmt"
	"net"
)

func main() {
	conn, err := net.Dial("unix", "/tmp/order-agent.sock")
	if err != nil {
		fmt.Println("dial unix failed:", err)
		return
	}
	defer conn.Close()

	fmt.Fprintln(conn, "PING")

	reply, err := bufio.NewReader(conn).ReadString('\n')
	if err != nil {
		fmt.Println("read failed:", err)
		return
	}
	fmt.Print(reply)
}
```

Linux/macOS：

```bash
go run server.go
go run client.go
```

Windows PowerShell 建议使用 WSL：

```powershell
wsl
go run server.go
go run client.go
```

## 三、关键参数/语法/代码结构

`net.Listen("unix", socketPath)`：

- 创建 Unix Socket 监听。
- 地址是文件路径，不是 `host:port`。

`os.Remove(socketPath)`：

- 启动前清理旧 socket 文件。
- 如果上次进程异常退出，旧文件可能还在。

`net.Dial("unix", socketPath)`：

- 客户端连接本机 socket 文件。
- 读写方式仍然是 `net.Conn`。

## 四、真实后端场景示例

订单服务旁边有一个本机 agent，用来接收健康探测或动态配置刷新：

- 管理命令连接 `/tmp/order-agent.sock`。
- 发送 `PING` 或 `RELOAD_CONFIG`。
- agent 返回结果。

这种方式不需要对外暴露管理端口，适合只允许本机访问的控制面操作。但命令仍要鉴权或依赖文件权限，不能因为是本机就完全信任输入。

## 五、注意点

- Unix Socket 文件要放在权限可控的位置。
- 服务异常退出后可能留下旧 socket 文件，启动前要清理。
- Windows 场景要确认运行时和部署环境是否支持。
- 不适合跨机器通信，也不适合作为通用公网接口。

## 六、常见误区

- 误区一：Unix Socket 天然安全。  
  它减少了网络暴露面，但仍要管理文件权限和命令权限。

- 误区二：忘记清理旧 socket 文件。  
  进程异常退出后再次启动可能失败。

- 误区三：把 Unix Socket 当成跨机器方案。  
  它是本机通信手段，跨机器仍应使用 TCP、HTTP、RPC 或消息队列。

## 七、本节达标标准

- 能解释 Unix Socket 与 TCP 监听的区别。
- 能在 Linux/macOS 或 WSL 中运行 Unix Socket 示例。
- 能说明为什么要清理旧 socket 文件。
- 能判断本机管理接口是否适合使用 Unix Socket。
