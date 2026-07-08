# 1. slog 结构化日志

本节目标：学会使用 Go 标准库 `log/slog` 输出结构化日志。

---

## 一、为什么需要结构化日志

普通日志：

```text
GET /todos 200 10ms
```

结构化日志：

```text
time=... level=INFO msg=request method=GET path=/todos status=200 duration=10ms
```

结构化日志更适合被日志平台检索、过滤和统计。

---

## 二、最小 slog 示例

```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

logger.Info("server started", "addr", ":8080")
logger.Error("server error", "error", err)
```

导入：

```go
import (
	"log/slog"
	"os"
)
```

---

## 三、JSON 日志

生产环境常用 JSON：

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
```

输出更适合日志采集系统处理。

---

## 四、在 Handler 中使用 logger

```go
type Handler struct {
	logger *slog.Logger
}

func (h *Handler) ListTodos(w http.ResponseWriter, r *http.Request) {
	h.logger.Info("list todos", "request_id", getRequestID(r))
}
```

不要在各处随意创建 logger。建议在程序启动时创建，然后注入到需要的结构体中。

---

## 五、本节检查点

请确认你能做到：

- 创建 `slog.Logger`。
- 输出文本日志和 JSON 日志。
- 使用键值字段记录上下文。
- 理解为什么结构化日志更适合生产排查。

