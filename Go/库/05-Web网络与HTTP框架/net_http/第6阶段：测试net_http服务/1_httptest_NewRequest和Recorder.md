# 1. httptest.NewRequest 和 ResponseRecorder

本节目标：学会在测试中构造 HTTP 请求，并记录 Handler 写出的响应。

---

## 一、为什么不用每次手动启动服务

手动测试流程是：

```text
go run .
curl ...
观察结果
```

这适合学习和调试，但不适合自动化测试。

自动化测试希望直接调用 Handler：

```text
构造请求
-> 调用 handler.ServeHTTP
-> 检查响应状态码、Header、Body
```

标准库 `net/http/httptest` 就是为这个场景准备的。

---

## 二、最小 Handler 测试

Handler：

```go
func health(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "ok")
}
```

测试：

```go
func TestHealth(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/health", nil)
	rec := httptest.NewRecorder()

	health(rec, req)

	resp := rec.Result()
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusOK)
	}

	body, _ := io.ReadAll(resp.Body)
	if string(body) != "ok\n" {
		t.Fatalf("body = %q, want %q", string(body), "ok\n")
	}
}
```

需要导入：

```go
import (
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"testing"
)
```

---

## 三、NewRequest 做了什么

```go
req := httptest.NewRequest(http.MethodGet, "/health", nil)
```

它创建一个测试用的 `*http.Request`。

参数：

- 请求方法。
- URL。
- 请求体。

如果是 JSON Body：

```go
body := strings.NewReader(`{"title":"learn"}`)
req := httptest.NewRequest(http.MethodPost, "/todos", body)
req.Header.Set("Content-Type", "application/json")
```

---

## 四、NewRecorder 做了什么

```go
rec := httptest.NewRecorder()
```

它实现了 `http.ResponseWriter`，可以记录 Handler 写出的：

- 状态码。
- Header。
- Body。

调用 Handler：

```go
handler.ServeHTTP(rec, req)
```

或者如果是普通函数：

```go
health(rec, req)
```

---

## 五、检查 JSON 响应

不要只比较整个 JSON 字符串，因为字段顺序、空格、换行可能不同。

推荐解析：

```go
var got Todo
if err := json.NewDecoder(resp.Body).Decode(&got); err != nil {
	t.Fatalf("decode response: %v", err)
}

if got.Title != "learn" {
	t.Fatalf("title = %q, want %q", got.Title, "learn")
}
```

---

## 六、本节检查点

请确认你能做到：

- 用 `httptest.NewRequest` 构造请求。
- 用 `httptest.NewRecorder` 记录响应。
- 检查状态码。
- 检查响应 Header。
- 解析并断言 JSON 响应。

