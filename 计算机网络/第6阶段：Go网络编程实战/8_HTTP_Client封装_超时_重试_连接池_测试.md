# 8. HTTP Client 封装：超时、重试、连接池与测试

本节目标：写一个可复用的 Go HTTP Client 封装，掌握请求级超时、连接池配置、响应体关闭、有限重试、错误分类和 `httptest` 测试。

Go 后端工程师经常要调用其他服务。很多线上事故不是不会发 HTTP 请求，而是没有处理好超时、重试和连接复用。

---

## 一、最终目标

完成后你会得到：

```text
http-client-lab/
  go.mod
  client/client.go
  client/client_test.go
```

它支持：

- 统一创建带连接池的 `http.Client`。
- 对单次请求设置 `context` 超时。
- 对 `GET` 请求做有限重试。
- 遇到 `5xx` 可以重试，遇到 `4xx` 不重试。
- 通过 `httptest.Server` 写可重复的单元测试。

---

## 二、创建项目

```powershell
mkdir http-client-lab
cd http-client-lab
go mod init example.com/http-client-lab
mkdir client
```

本节不依赖外部第三方库，全部使用标准库。

---

## 三、先写 Client 结构

创建文件：

```text
client/client.go
```

写入第一部分：

```go
package client

import (
    "context"
    "errors"
    "fmt"
    "io"
    "net"
    "net/http"
    "time"
)

type Client struct {
    httpClient *http.Client
    baseURL    string
    retries    int
}

type Option func(*Client)

func WithRetries(n int) Option {
    return func(c *Client) {
        if n >= 0 {
            c.retries = n
        }
    }
}
```

这里的设计是：

- `httpClient` 是真正发请求的标准库客户端。
- `baseURL` 是上游服务地址，例如 `http://user-service:8080`。
- `retries` 表示失败后最多重试几次。
- `Option` 是常见配置写法，后续可以继续加日志、Header、认证等配置。

---

## 四、配置 Transport 连接池

继续在 `client.go` 中写：

```go
func New(baseURL string, opts ...Option) *Client {
    transport := &http.Transport{
        Proxy: http.ProxyFromEnvironment,
        DialContext: (&net.Dialer{
            Timeout:   3 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        MaxIdleConns:          100,
        MaxIdleConnsPerHost:   20,
        IdleConnTimeout:       90 * time.Second,
        TLSHandshakeTimeout:   3 * time.Second,
        ExpectContinueTimeout: 1 * time.Second,
    }

    c := &Client{
        httpClient: &http.Client{
            Transport: transport,
        },
        baseURL: strings.TrimRight(baseURL, "/"),
        retries: 1,
    }

    for _, opt := range opts {
        opt(c)
    }

    return c
}
```

需要补一个 import：

```go
"strings"
```

这些参数要真正理解：

- `DialContext.Timeout`：TCP 建连最多等多久。
- `KeepAlive`：TCP keepalive 探测间隔，不等于 HTTP keep-alive。
- `MaxIdleConns`：所有 host 总共最多保留多少空闲连接。
- `MaxIdleConnsPerHost`：单个上游最多保留多少空闲连接。
- `IdleConnTimeout`：空闲连接保留多久。
- `TLSHandshakeTimeout`：TLS 握手最多等多久。

注意这里没有设置 `http.Client.Timeout`。原因是本节希望每个请求通过 `context.WithTimeout` 单独控制，这样不同接口可以有不同超时。

---

## 五、定义错误类型

继续写：

```go
var ErrUnexpectedStatus = errors.New("unexpected status")

type StatusError struct {
    StatusCode int
    Body       string
}

func (e *StatusError) Error() string {
    return fmt.Sprintf("%v: status=%d body=%s", ErrUnexpectedStatus, e.StatusCode, e.Body)
}

func (e *StatusError) Unwrap() error {
    return ErrUnexpectedStatus
}
```

为什么不用普通字符串错误？

因为调用方可能需要判断：

```go
var statusErr *StatusError
if errors.As(err, &statusErr) {
    // 根据 statusErr.StatusCode 做处理
}
```

这比从错误字符串里解析状态码可靠得多。

---

## 六、实现 GET 方法

继续写：

```go
func (c *Client) Get(ctx context.Context, path string, timeout time.Duration) ([]byte, error) {
    var lastErr error

    for attempt := 0; attempt <= c.retries; attempt++ {
        body, err := c.getOnce(ctx, path, timeout)
        if err == nil {
            return body, nil
        }

        lastErr = err
        if !shouldRetry(err) {
            return nil, err
        }

        if attempt < c.retries {
            timer := time.NewTimer(time.Duration(attempt+1) * 100 * time.Millisecond)
            select {
            case <-ctx.Done():
                timer.Stop()
                return nil, ctx.Err()
            case <-timer.C:
            }
        }
    }

    return nil, lastErr
}
```

这里的行为是：

- 第一次请求失败后，根据错误判断是否重试。
- 最多重试 `c.retries` 次。
- 重试前做一个很小的退避等待。
- 如果外层 `ctx` 已取消，立刻返回。

重试不能无限制。无限重试会把上游打得更严重，也会拖垮自己。

---

## 七、实现单次请求

继续写：

```go
func (c *Client) getOnce(parent context.Context, path string, timeout time.Duration) ([]byte, error) {
    ctx, cancel := context.WithTimeout(parent, timeout)
    defer cancel()

    url := c.baseURL + path
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
    if readErr != nil {
        return nil, readErr
    }

    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return nil, &StatusError{
            StatusCode: resp.StatusCode,
            Body:       string(data),
        }
    }

    return data, nil
}
```

几个细节非常重要：

- `NewRequestWithContext` 让请求可以被取消。
- `defer resp.Body.Close()` 必须写，否则连接无法回到连接池。
- `io.LimitReader` 防止上游返回超大响应撑爆内存。
- 非 `2xx` 不直接当成功。

很多初学者只写：

```go
resp, _ := http.Get(url)
body, _ := io.ReadAll(resp.Body)
```

这在学习 demo 里可以，在工程中问题很大：没有超时、没有关闭、没有错误处理、没有响应大小限制。

---

## 八、实现重试判断

继续写：

```go
func shouldRetry(err error) bool {
    if err == nil {
        return false
    }

    if errors.Is(err, context.Canceled) {
        return false
    }

    if errors.Is(err, context.DeadlineExceeded) {
        return true
    }

    var statusErr *StatusError
    if errors.As(err, &statusErr) {
        return statusErr.StatusCode >= 500
    }

    var netErr net.Error
    if errors.As(err, &netErr) {
        return true
    }

    return false
}
```

重试原则：

- `context.Canceled` 不重试，调用方已经取消。
- 超时可以重试，但要谨慎。
- `5xx` 表示上游服务端错误，可以有限重试。
- `4xx` 表示请求本身可能有问题，通常不重试。
- 网络临时错误可以重试。

后端面试里经常问：“哪些请求能重试？”你不能只回答“失败就重试”。要结合幂等性、状态码和错误类型。

---

## 九、写第一个测试：成功响应

创建文件：

```text
client/client_test.go
```

写入：

```go
package client

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"
)

func TestGetSuccess(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/hello" {
            t.Fatalf("unexpected path: %s", r.URL.Path)
        }
        _, _ = w.Write([]byte("world"))
    }))
    defer srv.Close()

    c := New(srv.URL)
    got, err := c.Get(context.Background(), "/hello", time.Second)
    if err != nil {
        t.Fatalf("Get() error = %v", err)
    }

    if string(got) != "world" {
        t.Fatalf("body = %q", got)
    }
}
```

`httptest.NewServer` 会启动一个真实 HTTP 服务，只是端口由 Go 自动分配。这样测试不依赖外网，也不会因为网络波动失败。

运行：

```powershell
go test ./...
```

---

## 十、测试 4xx 不重试

继续在测试文件中写：

```go
func TestGetClientErrorNoRetry(t *testing.T) {
    calls := 0
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        calls++
        http.Error(w, "bad request", http.StatusBadRequest)
    }))
    defer srv.Close()

    c := New(srv.URL, WithRetries(3))
    _, err := c.Get(context.Background(), "/bad", time.Second)
    if err == nil {
        t.Fatal("expected error")
    }

    if calls != 1 {
        t.Fatalf("calls = %d, want 1", calls)
    }
}
```

这个测试验证：

```text
400 Bad Request 不应该重试。
```

如果你把所有失败都重试，这个测试会失败。

---

## 十一、测试 5xx 会重试

继续写：

```go
func TestGetServerErrorRetryThenSuccess(t *testing.T) {
    calls := 0
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        calls++
        if calls == 1 {
            http.Error(w, "temporary error", http.StatusBadGateway)
            return
        }
        _, _ = w.Write([]byte("ok"))
    }))
    defer srv.Close()

    c := New(srv.URL, WithRetries(2))
    got, err := c.Get(context.Background(), "/retry", time.Second)
    if err != nil {
        t.Fatalf("Get() error = %v", err)
    }

    if string(got) != "ok" {
        t.Fatalf("body = %q", got)
    }

    if calls != 2 {
        t.Fatalf("calls = %d, want 2", calls)
    }
}
```

这个测试模拟第一次上游返回 `502`，第二次恢复正常。

---

## 十二、测试请求超时

继续写：

```go
func TestGetTimeout(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(200 * time.Millisecond)
        _, _ = w.Write([]byte("late"))
    }))
    defer srv.Close()

    c := New(srv.URL, WithRetries(0))
    _, err := c.Get(context.Background(), "/slow", 50*time.Millisecond)
    if err == nil {
        t.Fatal("expected timeout error")
    }
}
```

这里不是为了判断错误字符串，而是确认超时确实能打断请求。

运行：

```powershell
go test ./...
```

全部通过后，说明你的客户端至少具备了工程基本盘。

---

## 十三、常见问题

### 1. `http.Client` 能不能每次请求都 new 一个？

不建议。

`http.Client` 内部通过 `Transport` 管理连接池。每次请求都创建新 client，很容易导致连接无法复用，增加 TCP 握手和 TLS 握手成本。

推荐做法是：一个上游服务复用一个 client，或者复用一组配置相同的 client。

### 2. 为什么必须关闭 `resp.Body`？

因为只有读完并关闭响应体，连接才有机会回到连接池。

如果忘记关闭，表现可能是：

```text
连接数不断增长。
TIME_WAIT 增多。
请求越来越慢。
最终出现 too many open files。
```

### 3. 设置了 `context.WithTimeout`，还需要 Transport 的 Dial 超时吗？

需要。

请求超时控制的是整个请求生命周期，Dial 超时控制的是建连阶段。工程上通常两者都配置，便于定位问题发生在哪一段。

### 4. 所有 GET 都能重试吗？

通常 GET 更接近幂等，但也不能绝对化。

如果某个 GET 接口会触发副作用，例如统计、扣费、触发任务，那么重试也可能造成问题。工程上要结合接口语义判断。

### 5. POST 能重试吗？

默认不要随便重试。

除非它有幂等键，例如 `Idempotency-Key`，或者服务端明确保证重复请求不会造成重复写入。

---

## 十四、练习

请继续完成：

1. 给请求增加 `User-Agent`。
2. 给 `Client` 增加 `PostJSON` 方法。
3. 为 `PostJSON` 增加最大响应体限制。
4. 用 `errors.As` 在测试中断言 `StatusError.StatusCode`。
5. 打印每次重试的 attempt 编号。

这些练习做完后，你对 Go 调用上游服务的工程细节会更踏实。

---

## 十五、本节达标标准

学完本节后，你应该能够做到：

- 配置 `http.Transport` 的连接池参数。
- 解释 `DialContext.Timeout`、`TLSHandshakeTimeout`、`IdleConnTimeout` 的作用。
- 使用 `context.WithTimeout` 控制单次请求。
- 正确关闭 `resp.Body`。
- 对响应体大小做限制。
- 区分 `4xx`、`5xx`、网络错误和上下文取消。
- 编写有限重试逻辑。
- 使用 `httptest.Server` 测试 HTTP Client。
- 说明哪些请求适合重试，哪些请求不适合重试。

