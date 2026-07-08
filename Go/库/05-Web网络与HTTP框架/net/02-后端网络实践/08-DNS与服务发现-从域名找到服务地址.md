# 8. DNS 与服务发现：从域名找到服务地址

本节目标：学完后你能使用 Go 查询域名地址，并理解后端服务依赖 DNS 时要控制超时和降级。

简短引入：真实后端项目很少直接写死 IP。服务通常通过域名、Kubernetes Service、内网 DNS 或配置中心找到对方。DNS 解析失败，会让服务看起来像“网络不通”，但排查路径完全不同。

## 一、为什么需要它

常见场景：

- 订单服务访问 `inventory.service.local`。
- 邮件服务查询 MX 记录。
- 客户端 SDK 通过域名访问网关。
- 多机房部署通过 DNS 返回不同地址。

DNS 给我们带来灵活性，也带来缓存、超时、污染、解析失败等问题。

```text
域名不是地址本身；域名解析是一次可能失败、可能超时的网络操作。
```

## 二、基本用法

```go
package main

import (
	"context"
	"fmt"
	"net"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	resolver := net.Resolver{}
	addrs, err := resolver.LookupHost(ctx, "example.com")
	if err != nil {
		fmt.Println("lookup failed:", err)
		return
	}

	for _, addr := range addrs {
		fmt.Println(addr)
	}
}
```

运行：

```bash
go run .
```

## 三、关键参数/语法/代码结构

`net.LookupHost(host)`：

- 简单查询主机名。
- 不方便传 `context`，学习可以用，项目中更推荐 `Resolver`。

`resolver.LookupHost(ctx, host)`：

- 可以用 `context` 控制超时和取消。
- 更适合后端服务。

`net.LookupPort("tcp", "http")`：

- 可以把服务名解析成端口，例如 `http -> 80`。
- 真实项目中端口通常来自配置，不建议依赖服务名隐式解析。

## 四、真实后端场景示例

订单服务启动时检查库存服务域名：

```go
package main

import (
	"context"
	"fmt"
	"net"
	"time"
)

func main() {
	host := "inventory.service.local"

	ctx, cancel := context.WithTimeout(context.Background(), 1500*time.Millisecond)
	defer cancel()

	addrs, err := net.DefaultResolver.LookupHost(ctx, host)
	if err != nil {
		fmt.Println("dns check failed:", err)
		fmt.Println("service can still start, but downstream calls must handle failure")
		return
	}

	fmt.Println("inventory addresses:", addrs)
}
```

这里有一个保守建议：启动检查可以帮助早发现配置错误，但不要把它当作唯一保护。服务启动后 DNS 仍可能变化或失败，每次真正调用下游时都要有超时和错误处理。

在真实项目里，DNS 通常还会和服务发现配合使用。比如 Kubernetes 里访问 `inventory.default.svc.cluster.local`，表面上看是访问一个域名，背后可能对应多个 Pod 地址。你不需要在业务代码里手动管理这些地址，但要知道：当 Pod 扩缩容、节点故障、DNS 服务异常时，调用方看到的错误可能只是“解析失败”或“连接超时”。

可以给调用链路拆成三段记录：

```text
resolve -> dial -> request
```

这样日志里能区分：

- `resolve_failed`：域名解析失败，优先看 DNS、服务名、命名空间、网络策略。
- `dial_failed`：解析成功但连接失败，优先看端口、监听地址、防火墙、安全组。
- `request_failed`：连接成功但业务失败，优先看协议、参数、下游业务日志。

简单客户端调用可以按这个思路拆开：

```go
package main

import (
	"context"
	"fmt"
	"net"
	"time"
)

func main() {
	host := "example.com"
	port := "80"

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	addrs, err := net.DefaultResolver.LookupHost(ctx, host)
	if err != nil {
		fmt.Println("event=resolve_failed host="+host, "err=", err)
		return
	}
	fmt.Println("event=resolve_ok host="+host, "addrs=", addrs)

	dialer := net.Dialer{Timeout: 2 * time.Second}
	conn, err := dialer.DialContext(ctx, "tcp", net.JoinHostPort(host, port))
	if err != nil {
		fmt.Println("event=dial_failed host="+host, "port="+port, "err=", err)
		return
	}
	defer conn.Close()

	fmt.Println("event=dial_ok remote=" + conn.RemoteAddr().String())
}
```

## 五、注意点

- DNS 结果可能变化，不要自己做永久缓存。
- DNS 查询也需要超时，否则故障时会拖慢请求。
- 容器和 Kubernetes 环境里，DNS 故障是常见问题，要记录 host、耗时和错误。
- 如果是核心链路，下游地址变更要有发布和回滚方案。
- 不要把 DNS 解析结果写进配置文件长期固定，除非你非常确定地址不会变化。
- 客户端连接池可能会长期复用旧连接，DNS 变化不一定马上生效，要结合连接最大生命周期或重建连接策略。
- 如果服务名来自用户输入或外部参数，要防止 SSRF 风险，不要允许任意内网域名被访问。

## 六、常见误区

- 误区一：域名能 ping 通，服务就一定可用。  
  DNS 只说明能解析或能到达某个 IP，不代表端口、协议和业务健康。

- 误区二：启动时解析一次后永久使用。  
  服务扩缩容、故障转移后地址可能变化。

- 误区三：把 DNS 错误当成业务错误。  
  DNS 失败属于基础设施或配置问题，日志和告警要能区分。

- 误区四：为了“稳定”把域名解析成 IP 后写死。  
  短期可能绕过 DNS 问题，长期会破坏扩缩容、故障迁移和灰度发布。

## 七、本节达标标准

- 能使用 `Resolver.LookupHost` 查询域名。
- 能给 DNS 查询设置 `context` 超时。
- 能说明 DNS 成功不等于服务可用。
- 能在后端调用链路中把 DNS、连接、业务错误分开记录。
- 能根据 `resolve_failed`、`dial_failed`、`request_failed` 判断排查方向。
