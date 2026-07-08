# 1. 准备一个用于 Nginx 代理的 Go 服务

本节目标：写一个适合学习 Nginx 代理的 Go HTTP 服务。它要能返回 ping、打印请求头、模拟慢请求，后面所有代理实验都会用到它。

---

## 一、项目目录

创建：

```text
nginx-go-lab/
  app/
    go.mod
    main.go
```

进入目录：

```bash
mkdir -p ~/nginx-go-lab/app
cd ~/nginx-go-lab/app
go mod init nginx-go-lab
```

---

## 二、完整 Go 代码

创建 `main.go`：

```go
package main

import (
	"encoding/json"
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
			"remote_addr":       r.RemoteAddr,
			"host":              r.Host,
			"x_real_ip":         r.Header.Get("X-Real-IP"),
			"x_forwarded_for":   r.Header.Get("X-Forwarded-For"),
			"x_forwarded_proto": r.Header.Get("X-Forwarded-Proto"),
			"user_agent":        r.UserAgent(),
		})
	})

	mux.HandleFunc("/api/slow", func(w http.ResponseWriter, r *http.Request) {
		seconds := 5
		if raw := r.URL.Query().Get("seconds"); raw != "" {
			if n, err := strconv.Atoi(raw); err == nil {
				seconds = n
			}
		}

		time.Sleep(time.Duration(seconds) * time.Second)

		writeJSON(w, map[string]string{
			"message": "slow response finished",
			"port":    port,
		})
	})

	log.Printf("listening on :%s", port)
	log.Fatal(http.ListenAndServe(":"+port, logRequest(mux)))
}

func writeJSON(w http.ResponseWriter, v any) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(v)
}

func logRequest(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s remote=%s cost=%s", r.Method, r.URL.RequestURI(), r.RemoteAddr, time.Since(start))
	})
}
```

---

## 三、启动服务

```bash
go run main.go
```

默认监听：

```text
:8080
```

如果要指定端口：

```bash
PORT=8081 go run main.go
```

PowerShell：

```powershell
$env:PORT="8081"; go run main.go
```

---

## 四、直连验证

健康检查：

```bash
curl http://127.0.0.1:8080/health
```

ping：

```bash
curl http://127.0.0.1:8080/api/ping
```

请求头：

```bash
curl http://127.0.0.1:8080/api/headers
```

慢请求：

```bash
curl "http://127.0.0.1:8080/api/slow?seconds=3"
```

---

## 五、为什么要准备这些接口

### /api/ping

用于验证代理是否打通。

### /api/headers

用于观察 Nginx 是否正确传递代理头。

### /api/slow

用于制造 504，学习超时排查。

### /health

用于健康检查。真实项目中，负载均衡、容器编排、监控系统都会用到类似接口。

---

## 六、常见问题

### 1. 端口被占用

```text
bind: address already in use
```

检查：

```bash
ss -lntp | grep 8080
```

换端口：

```bash
PORT=8081 go run main.go
```

### 2. curl 连接失败

确认 Go 服务是否还在运行。

```bash
ps aux | grep main
```

### 3. 访问 Nginx 失败但直连 Go 成功

说明 Go 服务没问题，优先查 Nginx 配置、端口、日志。

---

## 七、本节练习

1. 启动 Go 服务。
2. 访问 `/api/ping`。
3. 访问 `/api/headers`。
4. 访问 `/api/slow?seconds=3`。
5. 用 `PORT=8081` 再启动一个实例。
6. 观察两个实例返回的 port 是否不同。

---

## 八、本节复盘

请确认你能回答：

1. 为什么要先直连 Go 验证？
2. `/api/headers` 后面用来观察什么？
3. `/api/slow` 后面用来制造什么问题？
4. 如何启动多个不同端口的 Go 实例？
5. 如果直连 Go 成功但 Nginx 访问失败，排查方向是什么？

