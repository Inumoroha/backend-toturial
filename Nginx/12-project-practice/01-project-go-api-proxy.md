# 01 项目实战：Go API 反向代理

## 项目目标

完成一个最小但完整的 Go API + Nginx 反向代理项目。

最终效果：

```text
Client -> Nginx(api.local:80) -> Go API(127.0.0.1:8080)
```

## 一、功能要求

Go 服务提供：

- `GET /api/ping`：返回 pong。
- `GET /api/headers`：返回代理头信息。
- `GET /health`：返回 ok。

Nginx 提供：

- 监听 `api.local:80`。
- 转发请求到 Go 服务。
- 设置真实 IP 和协议代理头。
- 配置基础超时。
- 记录访问日志。

## 二、Go 服务代码

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("ok"))
	})

	http.HandleFunc("/api/ping", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]string{"message": "pong"})
	})

	http.HandleFunc("/api/headers", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]string{
			"remote_addr":           r.RemoteAddr,
			"host":                  r.Host,
			"x_real_ip":             r.Header.Get("X-Real-IP"),
			"x_forwarded_for":       r.Header.Get("X-Forwarded-For"),
			"x_forwarded_proto":     r.Header.Get("X-Forwarded-Proto"),
			"x_forwarded_host":      r.Header.Get("X-Forwarded-Host"),
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

## 三、Nginx 配置

```nginx
server {
    listen 80;
    server_name api.local;

    access_log /var/log/nginx/api_access.log;
    error_log /var/log/nginx/api_error.log warn;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;

        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

## 四、验证命令

```bash
curl http://api.local/health
curl http://api.local/api/ping
curl http://api.local/api/headers
sudo tail -n 20 /var/log/nginx/api_access.log
```

## 五、验收标准

- `curl http://api.local/api/ping` 返回 `pong`。
- Go 服务能读取到 `X-Real-IP`。
- access log 中能看到请求记录。
- 停掉 Go 服务后，Nginx 返回 502，并且 error log 有记录。

## 六、复盘问题

1. 为什么 Go 服务看到的 `RemoteAddr` 不一定是真实用户 IP？
2. `X-Forwarded-For` 可能包含多个 IP，为什么？
3. 如果 Nginx 返回 502，你会按什么顺序排查？

