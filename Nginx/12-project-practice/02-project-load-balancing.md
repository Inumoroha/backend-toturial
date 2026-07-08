# 02 项目实战：三实例 Go API 负载均衡

## 项目目标

部署三个 Go API 实例，并使用 Nginx 进行负载均衡。

最终效果：

```text
Client -> Nginx -> Go API :8080
                -> Go API :8081
                -> Go API :8082
```

## 一、功能要求

每个 Go 实例返回自己的端口：

```json
{"message":"pong","port":"8080"}
```

Nginx 要求：

- 使用 `upstream go_api`。
- 支持三个后端实例。
- 配置代理头。
- 配置失败重试。

## 二、Go 服务代码

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
		json.NewEncoder(w).Encode(map[string]string{
			"message": "pong",
			"port":    port,
		})
	})

	log.Printf("listening on :%s", port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

启动：

```bash
PORT=8080 go run main.go
PORT=8081 go run main.go
PORT=8082 go run main.go
```

## 三、Nginx 配置

```nginx
upstream go_api {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8081 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=10s;
}

server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://go_api;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 3;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 四、验证命令

```bash
for i in {1..12}; do curl -s http://api.local/api/ping; echo; done
```

你应该看到不同端口轮流返回。

停止一个实例后再请求：

```bash
for i in {1..12}; do curl -s -o /dev/null -w "%{http_code}\n" http://api.local/api/ping; done
```

## 五、验收标准

- 请求能分发到多个 Go 实例。
- 停掉一个实例后，服务仍可用。
- error log 中能看到失败实例的连接错误。
- 你能解释 `max_fails`、`fail_timeout`、`proxy_next_upstream` 的作用。

## 六、复盘问题

1. 如果接口是创建订单，失败重试会有什么风险？
2. 为什么多实例服务应该尽量无状态？
3. 如果使用登录 session，你会把 session 存在哪里？

