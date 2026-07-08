# 7. Go HTTP Client：超时、连接池与请求取消

本节目标：掌握生产中 Go HTTP Client 的基本配置，避免无超时、连接泄漏、连接池配置不当。

---

## 一、不要长期使用默认写法

```go
resp, err := http.Get("https://example.com")
```

这适合临时脚本，不适合作为生产服务统一写法。

生产中你需要：

- 超时。
- 连接池。
- context 取消。
- 日志和指标。
- 错误处理。

---

## 二、推荐 Client

```go
client := &http.Client{
    Timeout: 5 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:          100,
        MaxIdleConnsPerHost:   20,
        MaxConnsPerHost:       100,
        IdleConnTimeout:       90 * time.Second,
        TLSHandshakeTimeout:   3 * time.Second,
        ResponseHeaderTimeout: 2 * time.Second,
    },
}
```

关键点：

- `http.Client` 应该复用。
- `Transport` 管理连接池。
- `Timeout` 控制整体请求时间。
- `ResponseHeaderTimeout` 控制等待响应头时间。

---

## 三、关闭响应体

```go
resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()
```

如果不关闭：

- 连接无法释放。
- 连接池无法复用。
- 可能出现 fd 泄漏。
- 可能出现大量 CLOSE_WAIT。

---

## 四、使用 context

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
    return err
}

resp, err := client.Do(req)
```

context 用于：

- 超时。
- 取消。
- 上游请求结束时取消下游请求。

---

## 补充实验：复现请求超时

先写一个慢服务：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

func main() {
    http.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(2 * time.Second)
        fmt.Fprintln(w, "done")
    })
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

再写客户端：

```go
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, "http://127.0.0.1:8080/slow", nil)
if err != nil {
    log.Fatal(err)
}

client := &http.Client{}
resp, err := client.Do(req)
if err != nil {
    log.Println("request failed:", err)
    return
}
defer resp.Body.Close()
```

预期错误中会包含：

```text
context deadline exceeded
```

这说明客户端没有无限等下去。线上调用下游服务时，每一层都应该有明确超时预算。

---

## 补充实践：读取并关闭响应体

如果你希望连接复用，推荐读取完响应体再关闭：

```go
resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()

body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
if err != nil {
    return err
}
_ = body
```

关键点：

```text
必须 Close。
最好限制最大读取大小。
读完 body 更有利于连接复用。
不要把无上限的响应直接 ReadAll 到内存。
```

这几个细节能避免连接泄漏、内存膨胀和连接池无法复用。

---

## 补充配置：按上游服务创建 Client

不要在项目里到处散落 `http.Client{}`。更推荐按上游封装：

```go
type UserClient struct {
    baseURL string
    client  *http.Client
}

func NewUserClient(baseURL string) *UserClient {
    return &UserClient{
        baseURL: baseURL,
        client: &http.Client{
            Transport: &http.Transport{
                MaxIdleConns:        100,
                MaxIdleConnsPerHost: 20,
                IdleConnTimeout:     90 * time.Second,
            },
        },
    }
}
```

好处：

```text
一个上游一套超时和连接池配置。
日志和指标可以按上游区分。
重试策略可以按接口语义配置。
测试时可以替换 baseURL 为 httptest.Server。
```

这就是从“会发请求”到“会管理下游依赖”的差别。

---

## 五、常见问题

### 1. 每次请求 new 一个 Client 可以吗？

不建议。会破坏连接复用，增加握手成本。

### 2. 只 Close 不读 body 可以吗？

必须 Close。是否读完影响连接复用，通常可以读完或丢弃后再关闭。

### 3. 重试是不是 Client 里默认有？

标准库不会自动对所有请求做业务重试。重试要结合幂等性和超时边界设计。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 写出带超时和连接池的 HTTP Client。
- 使用 context 控制请求取消。
- 正确关闭响应体。
- 解释为什么要复用 `http.Client`。
