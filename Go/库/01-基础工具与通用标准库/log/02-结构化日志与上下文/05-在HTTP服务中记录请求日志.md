# 05. 在 HTTP 服务中记录请求日志

本节目标：学完后，你能为 Go HTTP 服务编写请求日志中间件，记录方法、路径、状态码、耗时和 request_id。

## 简短引入

后端服务最常见的入口是 HTTP 请求。出了问题时，我们通常先问：哪个接口、哪个用户、哪个请求、耗时多久、返回什么状态码。请求日志中间件就是统一记录这些信息的位置。

## 一、为什么需要它

如果每个 handler 自己记录请求日志，会很容易遗漏字段。中间件可以把公共信息统一下来：

- 请求方法：GET、POST。
- 请求路径：`/articles`、`/orders`。
- 状态码：200、400、500。
- 耗时：用于排查慢接口。
- request_id：用于串联一次请求的多条日志。

```text
每个进入服务的请求，都应该有一个可以追踪的 request_id。
```

## 二、基本用法

下面是一个可运行 HTTP 服务。

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"strconv"
	"time"
)

type requestIDKey struct{}

type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

func accessLog(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		requestID := r.Header.Get("X-Request-Id")
		if requestID == "" {
			requestID = strconv.FormatInt(time.Now().UnixNano(), 10)
		}
		ctx := context.WithValue(r.Context(), requestIDKey{}, requestID)

		recorder := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(recorder, r.WithContext(ctx))

		logger.Info("http request completed",
			"request_id", requestID,
			"method", r.Method,
			"path", r.URL.Path,
			"status", recorder.status,
			"duration_ms", time.Since(start).Milliseconds(),
		)
	})
}

func requestIDFromContext(ctx context.Context) string {
	requestID, _ := ctx.Value(requestIDKey{}).(string)
	return requestID
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("/articles", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusCreated)
		_, _ = w.Write([]byte("article created"))
	})

	handler := accessLog(logger, mux)

	logger.Info("server started", "addr", ":8080")
	if err := http.ListenAndServe(":8080", handler); err != nil {
		logger.Error("server stopped", "error", err)
	}
}
```

运行服务：

```bash
go run .
```

另开终端请求。

Windows PowerShell：

```powershell
Invoke-WebRequest http://localhost:8080/articles -Method POST
```

Linux/macOS：

```bash
curl -X POST http://localhost:8080/articles
```

## 三、关键参数/语法/代码结构

`statusRecorder` 用来捕获响应状态码。普通的 `http.ResponseWriter` 写出状态后，我们不容易在中间件里知道最终状态，所以包一层记录下来。

`start := time.Now()` 用来计算请求耗时。真实项目中，`duration_ms` 是排查慢请求的重要字段。

`requestID` 优先从 `X-Request-Id` 读取。如果网关已经生成了请求 ID，服务应该继续沿用；如果没有，再由当前服务生成。示例里为了不引入第三方库，用时间戳生成，本地学习够用。生产项目通常会使用 UUID、雪花 ID 或网关统一生成的 trace id。

`context.WithValue` 这里的作用是把 request_id 放进请求上下文。这样 handler、service、repository 层都可以从 `r.Context()` 拿到同一个 request_id。

```text
request_id 一旦生成，就要沿着调用链传下去；只在请求日志里有，业务日志里没有，排查时仍然会断线。
```

## 四、真实后端场景示例

把 request_id 继续传给业务日志，可以这样做：

```go
func createArticleHandler(logger *slog.Logger) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		requestID := requestIDFromContext(r.Context())

		logger.Info("create article started",
			"request_id", requestID,
			"user_id", 1001,
			"title", "Go log guide",
		)

		w.WriteHeader(http.StatusCreated)
		_, _ = w.Write([]byte("ok"))
	}
}
```

真实项目里，service 层函数通常会接收 `context.Context`：

```go
func createArticle(ctx context.Context, logger *slog.Logger, userID int64, title string) error {
	logger.Info("create article in service",
		"request_id", requestIDFromContext(ctx),
		"user_id", userID,
	)
	return nil
}
```

这样做的好处是，后续访问数据库、调用缓存、发送消息时，都能继续带着同一个 request_id。排查线上问题时，你可以从一条请求日志追到业务日志，再追到数据库或外部依赖日志。

## 五、注意点

请求日志不要记录完整请求体。登录、支付、下单接口的请求体里可能有密码、Token、地址、手机号等敏感信息。

不要在中间件里做太重的逻辑。日志中间件应该稳定、轻量，不能因为记录日志拖慢业务接口。

状态码没有写入时默认是 200，因此 `statusRecorder` 初始化为 `http.StatusOK`。

## 六、常见误区

误区一：只在出错时记录请求。  
没有成功请求作为对比，很难判断错误是否集中在某个接口或某个时间段。

误区二：没有 request_id。  
一次请求可能产生多条日志，没有 request_id 就很难串起来。

误区三：记录完整请求体。  
这会带来敏感信息泄露和日志体积暴涨的问题。

## 七、本节达标标准

- 能编写 HTTP 请求日志中间件。
- 能记录 method、path、status、duration_ms。
- 知道 request_id 的作用。
- 知道请求体和敏感字段不能随意写入日志。
