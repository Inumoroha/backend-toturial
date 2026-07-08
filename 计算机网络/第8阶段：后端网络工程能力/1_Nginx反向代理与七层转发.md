# 1. Nginx 反向代理与七层转发

本节目标：理解反向代理的作用，并用 Nginx 代理本地 Go HTTP 服务。

---

## 一、反向代理是什么

```text
客户端 -> Nginx -> Go 服务
```

Nginx 代表后端接收请求，再转发给真正的 Go 服务。

它可以提供：

- 统一入口。
- TLS 终止。
- 路由转发。
- 访问日志。
- 负载均衡。
- 限流。

---

## 二、Nginx 配置示例

```nginx
server {
    listen 8080;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Go 服务监听 `8081`，客户端访问 Nginx 的 `8080`。

---

## 三、测试

启动 Go 服务：

```bash
PORT=8081 go run .
```

访问 Nginx：

```bash
curl -v http://127.0.0.1:8080
```

如果 Go 服务停止，Nginx 可能返回 502。

---

## 补充实验：最小 Go 服务 + Nginx 代理

先准备 Go 服务：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "hello from go, host=%s, remote=%s\n", r.Host, r.RemoteAddr)
    })

    log.Println("go backend listen on :8081")
    log.Fatal(http.ListenAndServe(":8081", nil))
}
```

启动：

```bash
go run .
```

Nginx 配置：

```nginx
server {
    listen 8080;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

验证：

```bash
curl -i http://127.0.0.1:8080/
```

你要确认：

```text
客户端访问的是 Nginx 8080。
Nginx 转发到 Go 8081。
Go 服务看到的 RemoteAddr 通常是 Nginx 的地址。
真实客户端 IP 需要从 X-Forwarded-For 读取。
```

---

## 补充排障：502 从哪里来

停止 Go 服务，再访问：

```bash
curl -i http://127.0.0.1:8080/
```

Nginx 通常返回：

```http
502 Bad Gateway
```

查看日志：

```bash
tail -f /var/log/nginx/error.log
```

常见错误：

```text
connect() failed (111: Connection refused) while connecting to upstream
```

这说明：

```text
Nginx 收到了客户端请求。
Nginx 尝试连接上游 Go 服务。
上游端口没有监听或连接失败。
```

所以 502 不等于“客户端到 Nginx 不通”，而是“Nginx 到上游出了问题”。

---

## 补充配置：代理超时要显式写

反向代理不要完全依赖默认超时。可以先理解这几个：

```nginx
proxy_connect_timeout 2s;
proxy_send_timeout 5s;
proxy_read_timeout 5s;
```

含义：

```text
proxy_connect_timeout：Nginx 连接上游最多等多久。
proxy_send_timeout：Nginx 向上游发送请求最多等多久。
proxy_read_timeout：Nginx 等待上游响应最多等多久。
```

如果 Go 服务处理需要 10 秒，但 Nginx `proxy_read_timeout` 是 5 秒，客户端会先看到网关超时。

这就是为什么后端服务、网关、客户端要有统一超时预算。

---

## 四、常见问题

### 1. 反向代理和正向代理区别？

正向代理代理客户端，反向代理代理服务端。

### 2. Nginx 看到的上游地址是什么？

由 `proxy_pass` 决定。

### 3. 后端如何获取真实客户端 IP？

通常从 `X-Forwarded-For` 或 `X-Real-IP` 获取，但只能信任可信代理设置的值。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 解释反向代理作用。
- 写出基础 Nginx `proxy_pass` 配置。
- 用 curl 验证代理转发。
- 解释 502 与上游不可用的关系。
