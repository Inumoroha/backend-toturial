# 05 项目实战：最终综合项目 Go 源码与运行说明

## 本节目标

给最终综合项目补齐可直接使用的 Go 服务源码。这个服务用于配合 Nginx 练习：

- 健康检查。
- ping 接口。
- 代理头检查。
- 慢请求制造 504。
- 上传接口制造 413 或验证大文件。
- 请求 ID 日志关联。
- 多实例返回端口，验证负载均衡。

## 一、项目目录

建议创建：

```text
nginx-capstone/
  app/
    go.mod
    main.go
  static/
    index.html
    assets/
      style.css
  nginx/
    conf.d/
      app.conf
```

## 二、go.mod

`app/go.mod`：

```go
module nginx-capstone-app

go 1.22
```

## 三、完整 main.go

`app/main.go`：

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"syscall"
	"time"
)

type app struct {
	port string
}

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	a := &app{port: port}

	mux := http.NewServeMux()
	mux.HandleFunc("/health", a.health)
	mux.HandleFunc("/api/ping", a.ping)
	mux.HandleFunc("/api/headers", a.headers)
	mux.HandleFunc("/api/slow", a.slow)
	mux.HandleFunc("/api/upload", a.upload)

	srv := &http.Server{
		Addr:         ":" + port,
		Handler:      requestLog(mux),
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 60 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	go func() {
		log.Printf("go api listening on :%s", port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("server error: %v", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	log.Println("shutdown signal received")
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		log.Fatalf("graceful shutdown failed: %v", err)
	}

	log.Println("server stopped")
}

func (a *app) health(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	_, _ = w.Write([]byte("ok\n"))
}

func (a *app) ping(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, map[string]string{
		"message": "pong",
		"port":    a.port,
	})
}

func (a *app) headers(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, map[string]string{
		"port":                a.port,
		"remote_addr":         r.RemoteAddr,
		"host":                r.Host,
		"x_request_id":        r.Header.Get("X-Request-ID"),
		"x_real_ip":           r.Header.Get("X-Real-IP"),
		"x_forwarded_for":     r.Header.Get("X-Forwarded-For"),
		"x_forwarded_proto":   r.Header.Get("X-Forwarded-Proto"),
		"user_agent":          r.UserAgent(),
	})
}

func (a *app) slow(w http.ResponseWriter, r *http.Request) {
	seconds := 5
	if raw := r.URL.Query().Get("seconds"); raw != "" {
		n, err := strconv.Atoi(raw)
		if err == nil && n >= 0 && n <= 120 {
			seconds = n
		}
	}

	time.Sleep(time.Duration(seconds) * time.Second)

	writeJSON(w, map[string]string{
		"message": fmt.Sprintf("slept %d seconds", seconds),
		"port":    a.port,
	})
}

func (a *app) upload(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	const maxRead = 200 << 20
	r.Body = http.MaxBytesReader(w, r.Body, maxRead)

	n, err := io.Copy(io.Discard, r.Body)
	if err != nil {
		http.Error(w, "failed to read body: "+err.Error(), http.StatusBadRequest)
		return
	}

	writeJSON(w, map[string]any{
		"message": "upload received",
		"bytes":   n,
		"port":    a.port,
	})
}

func requestLog(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		requestID := r.Header.Get("X-Request-ID")
		if requestID == "" {
			requestID = "missing"
		}

		next.ServeHTTP(w, r)

		log.Printf(
			"request_id=%s method=%s path=%s remote=%s cost=%s",
			requestID,
			r.Method,
			r.URL.RequestURI(),
			r.RemoteAddr,
			time.Since(start),
		)
	})
}

func writeJSON(w http.ResponseWriter, v any) {
	w.Header().Set("Content-Type", "application/json")
	enc := json.NewEncoder(w)
	enc.SetIndent("", "  ")
	_ = enc.Encode(v)
}
```

## 四、静态页面

`static/index.html`：

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Nginx Capstone</title>
  <link rel="stylesheet" href="/assets/style.css">
</head>
<body>
  <main>
    <h1>Nginx Capstone</h1>
    <p>静态页面由 Nginx 提供，API 请求由 Nginx 转发到 Go 多实例服务。</p>
  </main>
</body>
</html>
```

`static/assets/style.css`：

```css
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f6f8fa;
  color: #1f2328;
}

main {
  max-width: 760px;
  margin: 80px auto;
  padding: 0 24px;
}
```

## 五、运行三个 Go 实例

进入 `app` 目录：

```bash
go mod tidy
```

打开三个终端：

```bash
PORT=8080 go run main.go
```

```bash
PORT=8081 go run main.go
```

```bash
PORT=8082 go run main.go
```

验证：

```bash
curl http://127.0.0.1:8080/api/ping
curl http://127.0.0.1:8081/api/ping
curl http://127.0.0.1:8082/api/ping
```

## 六、Nginx 必备代理头

最终项目中，`/api/` 至少应传：

```nginx
proxy_set_header X-Request-ID $request_id;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto https;
```

验证：

```bash
curl -k https://app.local/api/headers
```

你应该能看到请求 ID、真实 IP、协议等信息。

## 七、测试慢请求

请求：

```bash
curl -k "https://app.local/api/slow?seconds=5"
```

制造 504：

```bash
curl -k "https://app.local/api/slow?seconds=40"
```

前提是 Nginx 中：

```nginx
proxy_read_timeout 30s;
```

然后查看：

```bash
sudo tail -n 30 /var/log/nginx/error.log
sudo tail -n 30 /var/log/nginx/access.log
```

## 八、测试上传

创建 20MB 文件：

```bash
dd if=/dev/zero of=/tmp/20m.bin bs=1M count=20
```

上传：

```bash
curl -k -X POST --data-binary @/tmp/20m.bin https://app.local/api/upload
```

如果普通 API 限制是 `10m`，但 `/api/upload` 是 `100m`，上传接口应该成功。

创建 120MB 文件：

```bash
dd if=/dev/zero of=/tmp/120m.bin bs=1M count=120
```

再上传应被 Nginx 或 Go 限制。

## 九、测试负载均衡

```bash
for i in {1..12}; do curl -k -s https://app.local/api/ping; echo; done
```

观察返回的 `port` 是否在 `8080`、`8081`、`8082` 之间变化。

## 十、测试请求 ID 关联

请求：

```bash
curl -k https://app.local/api/ping
```

查看 Nginx 日志中的 `rid=`，再查看 Go 日志中的 `request_id=`。两边应该一致。

## 十一、最终验收

完成后，你应该能演示：

- 静态首页由 Nginx 返回。
- `/assets/` 有缓存头。
- `/api/ping` 经过 upstream 负载均衡。
- `/api/headers` 能看到代理头。
- `/api/slow?seconds=40` 能复现 504。
- 停掉所有 Go 实例能复现 502。
- 高频请求能触发 429。
- `/api/upload` 有独立上传大小限制。
- Nginx 日志和 Go 日志能通过请求 ID 关联。

