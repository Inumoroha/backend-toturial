# 5. HTTP Client 连接池、context 与错误处理

本节目标：写出适合后端服务使用的 HTTP Client，掌握连接池、超时、context、响应体关闭和错误处理。

---

## 一、Client 应该复用

```go
var client = &http.Client{
    Timeout: 5 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
    },
}
```

不要每次请求都 new 一个 Client。否则连接池无法复用。

---

## 二、请求示例

```go
func fetch(ctx context.Context, url string) error {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return err
    }

    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode >= 500 {
        return fmt.Errorf("upstream status: %d", resp.StatusCode)
    }

    _, err = io.Copy(io.Discard, resp.Body)
    return err
}
```

---

## 三、为什么要读完并关闭 body

关闭 body 是必须的。

读完 body 有助于连接复用。否则连接可能无法回到连接池。

---

## 四、错误分类

常见错误来源：

- DNS 失败。
- TCP 连接失败。
- TLS 失败。
- context 超时。
- 服务返回 5xx。
- 读取 body 失败。

日志中最好区分这些阶段。

---

## 补充实验：忘记关闭 body 的后果

写一个循环请求：

```go
for i := 0; i < 1000; i++ {
    resp, err := client.Get("http://127.0.0.1:8080")
    if err != nil {
        log.Println(err)
        continue
    }
    _ = resp
}
```

这段代码故意没有关闭 `resp.Body`。

观察：

```bash
lsof -p 进程ID | wc -l
ss -antp | grep 8080 | wc -l
```

然后改成：

```go
resp, err := client.Get("http://127.0.0.1:8080")
if err != nil {
    return err
}
defer resp.Body.Close()
_, _ = io.Copy(io.Discard, resp.Body)
```

对比 fd 和连接数变化。

---

## 补充错误分类：不要只返回 err.Error

HTTP Client 错误至少分几类：

```text
DNS 错误。
TCP 建连错误。
TLS 错误。
context deadline exceeded。
HTTP 4xx。
HTTP 5xx。
响应体读取错误。
```

不同错误的处理方式不同。比如 `400` 不应该重试，`502` 可以有限重试，`context.Canceled` 说明上游调用可能是被调用方取消。

---

## 补充检查：每个上游都要有调用规范

为每个上游写清楚：

```text
baseURL。
单次请求超时。
是否允许重试。
哪些状态码重试。
最大响应体大小。
是否需要认证 Header。
日志字段。
```

示例：

```text
user-service：
timeout=800ms
retry=1
retry_on=502,503,504
max_body=1MB
```

有了这张表，代码里的 Client 配置就不是拍脑袋。

---

## 五、常见问题

### 1. Client.Timeout 和 context timeout 同时设置会怎样？

谁先到期谁生效。通常入口请求 context 用于链路取消，Client.Timeout 做兜底。

### 2. 状态码 500 会让 client.Do 返回 error 吗？

不会。HTTP 500 是正常收到响应，err 通常为 nil，需要自己检查状态码。

### 3. 什么时候重试？

只对幂等请求和短暂错误谨慎重试，并设置最大次数和退避。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 复用 HTTP Client。
- 配置连接池。
- 使用 context 控制取消。
- 正确关闭响应体。
- 区分网络错误和 HTTP 状态码错误。
