# 01 反向代理 Go 服务：proxy_pass 入门

## 本节目标

掌握 Nginx 反向代理 Go 服务的最基础配置。对 Go 后端工程师来说，这是 Nginx 最重要的能力。

## 一、反向代理是什么

客户端不会直接访问 Go 服务，而是访问 Nginx：

```text
Client -> Nginx -> Go API
```

好处：

- Go 服务不必直接暴露公网端口。
- Nginx 可以统一处理 HTTPS。
- Nginx 可以做负载均衡。
- Nginx 可以记录统一访问日志。
- Nginx 可以做限流、缓存、访问控制。

## 二、准备 Go API

创建或使用下面的 Go 服务：

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/api/ping", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{
			"message": "pong",
			"remote":  r.RemoteAddr,
			"host":    r.Host,
		})
	})

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

运行：

```bash
go run main.go
```

直接访问：

```bash
curl http://127.0.0.1:8080/api/ping
```

## 三、配置 Nginx 代理

```nginx
server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

通过 Nginx 访问：

```bash
curl http://api.local/api/ping
```

如果成功返回 JSON，说明代理链路打通。

## 四、proxy_pass 后面有没有斜杠

这是 Nginx 新手最容易踩的坑。

### 示例 1：不带 URI

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

请求：

```text
/api/ping
```

转发给后端仍然是：

```text
/api/ping
```

### 示例 2：带 `/`

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

请求：

```text
/api/ping
```

转发给后端变成：

```text
/ping
```

因为 `/api/` 这一段被替换成了 `/`。

## 五、建议规则

学习阶段先使用最不容易迷惑的写法：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

并让 Go 后端也使用 `/api/` 前缀。

如果你明确想让 Nginx 去掉 `/api/` 前缀，再使用：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

## 六、验证代理链路

使用：

```bash
curl -v http://api.local/api/ping
```

你要观察：

- 是否连接到了 Nginx。
- HTTP 状态码是多少。
- 响应体是否来自 Go 服务。
- Go 服务日志是否打印了请求。

## 七、本节练习

1. 启动 Go 服务，监听 `8080`。
2. 配置 Nginx 把 `api.local` 代理到 Go。
3. 使用 `curl` 访问 `/api/ping`。
4. 对比 `proxy_pass http://127.0.0.1:8080;` 和 `proxy_pass http://127.0.0.1:8080/;`。
5. 修改 Go 路由，观察 Nginx 路径转发的变化。

## 八、你应该掌握

学完本节，你应该能解释：

- 什么是反向代理。
- Nginx 为什么放在 Go 服务前面。
- `proxy_pass` 后面带不带 `/` 的区别。

