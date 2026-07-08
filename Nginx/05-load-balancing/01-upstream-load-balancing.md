# 01 负载均衡：upstream 与多 Go 实例

## 本节目标

学习 Nginx 如何把请求分发到多个 Go 服务实例。后端服务做横向扩展时，这是基础能力。

## 一、准备多个 Go 实例

修改 Go 服务，让它返回当前实例端口：

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	http.HandleFunc("/api/ping", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{
			"message": "pong",
			"port":    port,
		})
	})

	log.Printf("listening on :%s", port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

启动三个实例：

```bash
PORT=8080 go run main.go
PORT=8081 go run main.go
PORT=8082 go run main.go
```

如果在同一个终端不方便，可以开三个终端。

## 二、配置 upstream

```nginx
upstream go_api {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://go_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

验证：

```bash
for i in {1..10}; do curl -s http://api.local/api/ping; echo; done
```

你应该能看到请求被分发到不同端口。

## 三、默认轮询

Nginx 默认使用轮询方式：

```text
8080 -> 8081 -> 8082 -> 8080 -> ...
```

在请求耗时差不多、实例性能差不多的情况下，轮询简单有效。

## 四、权重配置

如果某台机器性能更强，可以设置更高权重：

```nginx
upstream go_api {
    server 127.0.0.1:8080 weight=3;
    server 127.0.0.1:8081 weight=1;
    server 127.0.0.1:8082 weight=1;
}
```

大致效果：`8080` 会接收更多请求。

## 五、最少连接

```nginx
upstream go_api {
    least_conn;

    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}
```

`least_conn` 会优先把请求分配给当前连接数较少的实例。接口耗时差异较大时，它可能比轮询更合适。

## 六、IP Hash

```nginx
upstream go_api {
    ip_hash;

    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}
```

同一个客户端 IP 会尽量落到同一个后端实例。

注意：现代后端通常更推荐无状态服务，把 session 放到 Redis、数据库或 token 中，而不是依赖 IP Hash。

## 七、本节练习

1. 启动 3 个 Go 实例。
2. 配置 `upstream go_api`。
3. 连续请求 10 次，观察流量分发。
4. 给 `8080` 设置更高权重，观察变化。
5. 尝试 `least_conn` 和 `ip_hash`。

## 八、你应该掌握

学完本节，你应该能解释：

- `upstream` 是什么。
- Nginx 如何把请求分发到多个 Go 实例。
- 轮询、权重、最少连接、IP Hash 分别适合什么场景。

