# 2. Go 服务实现

本节目标：实现最终项目使用的 Go API 服务。这个服务不追求复杂业务，而是专门用来验证 Nginx 的反向代理、负载均衡、日志、超时和上传配置。

---

## 一、创建项目

```bash
mkdir -p nginx-gateway-demo/app
cd nginx-gateway-demo/app
go mod init nginx-gateway-demo
```

---

## 二、接口回顾

需要实现：

- `GET /health`
- `GET /api/ping`
- `GET /api/headers`
- `GET /api/slow?seconds=5`
- `POST /api/upload`

每个实例要返回自己的端口，方便验证负载均衡。

---

## 三、完整代码

创建 `main.go`：

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	mux := http.NewServeMux()

	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ok\n"))
	})

	mux.HandleFunc("/api/ping", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, map[string]string{
			"message": "pong",
			"port":    port,
		})
	})

	mux.HandleFunc("/api/headers", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, map[string]string{
			"port":                port,
			"remote_addr":         r.RemoteAddr,
			"host":                r.Host,
			"x_request_id":        r.Header.Get("X-Request-ID"),
			"x_real_ip":           r.Header.Get("X-Real-IP"),
			"x_forwarded_for":     r.Header.Get("X-Forwarded-For"),
			"x_forwarded_proto":   r.Header.Get("X-Forwarded-Proto"),
		})
	})

	mux.HandleFunc("/api/slow", func(w http.ResponseWriter, r *http.Request) {
		seconds := 5
		if raw := r.URL.Query().Get("seconds"); raw != "" {
			if n, err := strconv.Atoi(raw); err == nil && n >= 0 && n <= 120 {
				seconds = n
			}
		}

		time.Sleep(time.Duration(seconds) * time.Second)

		writeJSON(w, map[string]string{
			"message": fmt.Sprintf("slept %d seconds", seconds),
			"port":    port,
		})
	})

	mux.HandleFunc("/api/upload", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}

		n, err := io.Copy(io.Discard, r.Body)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

		writeJSON(w, map[string]any{
			"message": "upload received",
			"bytes":   n,
			"port":    port,
		})
	})

	server := &http.Server{
		Addr:         ":" + port,
		Handler:      logRequest(mux),
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 90 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	log.Printf("listening on :%s", port)
	log.Fatal(server.ListenAndServe())
}

func writeJSON(w http.ResponseWriter, v any) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(v)
}

func logRequest(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf(
			"request_id=%s method=%s path=%s remote=%s cost=%s",
			r.Header.Get("X-Request-ID"),
			r.Method,
			r.URL.RequestURI(),
			r.RemoteAddr,
			time.Since(start),
		)
	})
}
```

---

## 四、运行单实例

```bash
go run main.go
```

验证：

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/api/ping
curl http://127.0.0.1:8080/api/headers
```

---

## 五、运行三实例

终端 1：

```bash
PORT=8080 go run main.go
```

终端 2：

```bash
PORT=8081 go run main.go
```

终端 3：

```bash
PORT=8082 go run main.go
```

验证：

```bash
curl http://127.0.0.1:8080/api/ping
curl http://127.0.0.1:8081/api/ping
curl http://127.0.0.1:8082/api/ping
```

---

## 六、测试慢接口

```bash
curl "http://127.0.0.1:8080/api/slow?seconds=3"
```

后面通过 Nginx 设置：

```nginx
proxy_read_timeout 30s;
```

再请求：

```bash
curl "https://app.local/api/slow?seconds=40"
```

即可复现 504。

---

## 七、测试上传接口

创建测试文件：

```bash
dd if=/dev/zero of=/tmp/1m.bin bs=1M count=1
```

上传：

```bash
curl -X POST --data-binary @/tmp/1m.bin http://127.0.0.1:8080/api/upload
```

---

## 八、本节练习

1. 启动单实例。
2. 验证所有接口。
3. 启动三实例。
4. 观察不同端口返回。
5. 请求慢接口，观察 Go 日志耗时。
6. 上传 1MB 文件，确认 bytes 返回正确。

---

## 九、本节复盘

请确认你能回答：

1. 为什么每个响应都要返回 port？
2. `/api/headers` 用来验证哪些代理头？
3. `/api/slow` 如何帮助复现 504？
4. `/api/upload` 如何帮助验证 Nginx body size？
5. Go 日志中的 request_id 来自哪里？

