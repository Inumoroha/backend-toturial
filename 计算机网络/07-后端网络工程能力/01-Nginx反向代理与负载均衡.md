# 01-Nginx 反向代理与负载均衡

## 学习目标

理解正向代理、反向代理、四层负载均衡、七层负载均衡，并能用 Nginx 代理多个 Go 服务实例。

## 一、正向代理与反向代理

正向代理代理客户端：

```text
客户端 -> 代理 -> 目标网站
```

反向代理代理服务端：

```text
客户端 -> Nginx -> Go 服务
```

后端工程中常说的网关、入口、负载均衡，大多是反向代理。

## 二、为什么需要反向代理

反向代理可以提供：

- 统一入口。
- TLS 终止。
- 路由转发。
- 负载均衡。
- 限流。
- 静态资源服务。
- 访问日志。
- 屏蔽后端实例细节。

## 三、四层与七层负载均衡

四层负载均衡：

- 基于 IP 和端口转发。
- 不理解 HTTP 内容。
- 性能高。

七层负载均衡：

- 基于 HTTP 路径、Header、Host 等转发。
- 能做更细粒度路由。
- 常用于 API 网关和 Ingress。

## 四、准备两个 Go 服务

服务代码：

```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    name := os.Getenv("APP_NAME")
    port := os.Getenv("PORT")
    if port == "" {
        port = "8081"
    }

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "hello from %s\n", name)
    })

    http.ListenAndServe(":"+port, nil)
}
```

启动两个实例：

```bash
APP_NAME=app-1 PORT=8081 go run .
APP_NAME=app-2 PORT=8082 go run .
```

## 五、Nginx 配置

```nginx
upstream go_backend {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 8080;

    location / {
        proxy_pass http://go_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Request-Id $request_id;
    }
}
```

测试：

```bash
curl http://127.0.0.1:8080
```

多请求几次，观察是否打到不同实例。

## 六、常见 Nginx 超时

```nginx
proxy_connect_timeout 2s;
proxy_send_timeout 5s;
proxy_read_timeout 10s;
```

含义：

- `proxy_connect_timeout`：连接上游超时。
- `proxy_send_timeout`：向上游发送请求超时。
- `proxy_read_timeout`：等待上游响应超时。

如果 `proxy_read_timeout` 超时，客户端常见 504。

## 七、Go 服务如何读取真实 IP

经过代理后，Go 服务看到的 `RemoteAddr` 通常是 Nginx 地址。真实客户端 IP 可能在：

```text
X-Forwarded-For
X-Real-IP
```

注意：这些 Header 可以被伪造，生产环境要只信任可信代理设置的值。

## 八、练习题

1. 正向代理和反向代理的区别是什么？
2. 四层负载均衡和七层负载均衡有什么区别？
3. Nginx 504 通常和哪个超时配置有关？
4. 为什么后端看到的客户端 IP 可能不是真实 IP？

## 九、验收标准

你能用 Nginx 代理两个 Go 服务，并解释请求如何被转发、如何负载均衡、如何保留客户端信息。

