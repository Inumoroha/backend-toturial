# 实现可诊断的 Go HTTP 服务

## 1. 创建项目

```bash
mkdir -p prometheus-lab/app/cmd/server
cd prometheus-lab/app
go mod init example.com/order-service
```

推荐从简单目录开始：

```text
app/
├── cmd/server/main.go
├── internal/httpapi/handler.go
├── internal/store/memory.go
├── internal/app/app.go
└── go.mod
```

先保持依赖少，理解标准库后再引入 Web 框架。

## 2. 服务代码的关键职责

一个后端服务至少要明确以下边界：

- `main` 负责读取配置、创建依赖、启动 server、等待退出信号。
- handler 负责协议层：参数校验、状态码和响应格式。
- service/use-case 负责业务流程。
- repository/client 负责外部依赖。
- middleware 负责 request id、超时、日志和后续指标。
- health handler 只表达服务是否适合接收流量，不要把所有深度检查都塞进 liveness。

不要把所有代码放进 `main.go`。Prometheus 埋点最终会进入 middleware 和依赖客户端，尽早分层可以避免指标重复统计。

## 3. 最小服务骨架

下面代码展示核心结构；生产项目应把处理器拆成独立文件并补齐错误日志。

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

type orderRequest struct {
	ProductID string `json:"product_id"`
	Quantity  int    `json:"quantity"`
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		_ = json.NewEncoder(w).Encode(map[string]string{"message": "hello"})
	})
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ok\n"))
	})
	mux.HandleFunc("POST /orders", func(w http.ResponseWriter, r *http.Request) {
		var req orderRequest
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil || req.ProductID == "" || req.Quantity <= 0 {
			http.Error(w, "invalid request", http.StatusBadRequest)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		_ = json.NewEncoder(w).Encode(map[string]string{"status": "created"})
	})

	srv := &http.Server{
		Addr:              ":8080",
		Handler:           requestLog(logger, mux),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      10 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	go func() {
		logger.Info("server_started", "addr", srv.Addr)
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("server_failed", "error", err)
			os.Exit(1)
		}
	}()

	<-ctx.Done()
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		logger.Error("shutdown_failed", "error", err)
		os.Exit(1)
	}
	logger.Info("server_stopped")
}

func requestLog(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		logger.Info("request_finished",
			"method", r.Method,
			"path", r.URL.Path,
			"duration_ms", time.Since(start).Milliseconds(),
		)
	})
}
```

### 为什么要设置这些 timeout

- `ReadHeaderTimeout` 防止客户端慢慢发送请求头占住连接。
- `ReadTimeout` 限制读取请求体的最长时间。
- `WriteTimeout` 限制服务端写响应的时间。
- `IdleTimeout` 限制 keep-alive 空闲连接。
- `Shutdown` 给正在处理的请求一个有限的完成窗口。

超时不是越短越好。要根据请求体大小、下游超时和用户体验设置，并把超时作为后续指标和告警的候选原因。

## 4. 测试请求行为

使用 `httptest.NewServer` 或 `httptest.NewRecorder`：

```go
func TestCreateOrderRejectsInvalidQuantity(t *testing.T) {
	req := httptest.NewRequest(http.MethodPost, "/orders", strings.NewReader(`{"product_id":"p1","quantity":0}`))
	rec := httptest.NewRecorder()

	handler := buildHandler()
	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusBadRequest {
		t.Fatalf("status = %d, want %d", rec.Code, http.StatusBadRequest)
	}
}
```

至少覆盖：

- 合法创建返回 201。
- JSON 格式错误返回 400。
- 数量为 0 或负数返回 400。
- 依赖超时返回 503 或项目约定的错误码。
- context 取消后不会继续等待依赖。
- health endpoint 的状态符合部署探针约定。

## 5. 加入可控故障注入

为了后续学习指标和告警，增加仅在本地启用的配置：

```text
INJECT_DELAY_MS=0
INJECT_ERROR_RATE=0
```

注意：

- 随机错误应该使用独立随机源，便于测试。
- 配置必须明确标注为实验能力，生产默认关闭。
- 不要在指标 label 中记录每次随机错误的详细文本。
- 故障注入接口不能暴露到公网。

## 6. 性能和诊断命令

```bash
go test ./...
go test -race ./...
go test -bench . -benchmem ./...
go vet ./...
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30
```

你要记录的不只是“命令通过”，还包括：

- benchmark 的吞吐、延迟和分配次数。
- race detector 是否发现问题。
- pprof 中最耗 CPU 的函数。
- 在注入延迟后，客户端和服务端各自的等待时间。

## 阶段练习

1. 为订单接口增加一个模拟库存依赖。
2. 给依赖调用加 500ms 超时。
3. 使用 context 将请求取消传递给依赖。
4. 写测试验证依赖超时时返回 503。
5. 用 `curl` 连续发起请求，保存服务日志。
6. 为每个状态码写一句面向运维的解释。

## 完成标准

你能从服务日志中找到一次请求，并回答：请求何时开始、命中了哪个路由、调用了哪个依赖、为什么返回这个状态码、总耗时是多少。这个答案将由后续 Metrics 和 Tracing 逐步自动化。

