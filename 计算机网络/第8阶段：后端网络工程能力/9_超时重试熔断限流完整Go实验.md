# 9. 超时、重试、熔断、限流完整 Go 实验

本节目标：用一个可运行的 Go 项目观察超时、重试、熔断和限流如何共同影响后端服务稳定性。

这些词经常一起出现，但它们解决的问题不同：

```text
超时：不要无限等。
重试：临时失败时再试一次。
熔断：连续失败时先别打上游。
限流：入口流量太大时主动拒绝。
```

如果它们配置不当，也会互相伤害。例如无限重试会放大故障，过长超时会拖垮线程和连接，过激熔断会误伤正常流量。

---

## 一、最终目标

项目目录：

```text
resilience-lab/
  go.mod
  cmd/upstream/main.go
  cmd/api/main.go
```

运行后：

```text
upstream: 127.0.0.1:9001
api:      127.0.0.1:8080
```

调用：

```text
GET /proxy
```

API 服务会调用 upstream，并应用：

- 每次上游调用 500ms 超时。
- 最多重试 1 次。
- 连续失败 3 次后熔断 5 秒。
- 每秒最多接收 5 个入口请求。

---

## 二、创建项目

```powershell
mkdir resilience-lab
cd resilience-lab
go mod init example.com/resilience-lab
mkdir cmd
mkdir cmd\upstream
mkdir cmd\api
```

本节不引入第三方库，先手写最小版本，方便理解原理。

---

## 三、实现不稳定上游

创建：

```text
cmd/upstream/main.go
```

写入：

```go
package main

import (
    "encoding/json"
    "flag"
    "log"
    "math/rand"
    "net/http"
    "time"
)

func main() {
    addr := flag.String("addr", ":9001", "listen address")
    slow := flag.Duration("slow", 0, "extra delay")
    failRate := flag.Float64("fail", 0, "fail rate, 0~1")
    flag.Parse()

    mux := http.NewServeMux()
    mux.HandleFunc("GET /data", func(w http.ResponseWriter, r *http.Request) {
        if *slow > 0 {
            time.Sleep(*slow)
        }
        if *failRate > 0 && rand.Float64() < *failRate {
            http.Error(w, "temporary upstream error", http.StatusBadGateway)
            return
        }

        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(map[string]string{
            "message": "hello from upstream",
        })
    })

    srv := &http.Server{
        Addr:              *addr,
        Handler:           mux,
        ReadHeaderTimeout: 3 * time.Second,
    }

    log.Printf("upstream listening on %s", *addr)
    log.Fatal(srv.ListenAndServe())
}
```

这个上游可以模拟两类问题：

```text
-slow 1s     响应变慢。
-fail 0.5    50% 概率返回 502。
```

---

## 四、实现固定窗口限流器

创建：

```text
cmd/api/main.go
```

先写限流器：

```go
package main

import (
    "context"
    "errors"
    "io"
    "log"
    "net"
    "net/http"
    "sync"
    "time"
)

type limiter struct {
    mu          sync.Mutex
    windowStart time.Time
    count       int
    limit       int
}

func newLimiter(limit int) *limiter {
    return &limiter{limit: limit}
}

func (l *limiter) allow() bool {
    l.mu.Lock()
    defer l.mu.Unlock()

    now := time.Now()
    if l.windowStart.IsZero() || now.Sub(l.windowStart) >= time.Second {
        l.windowStart = now
        l.count = 1
        return true
    }

    if l.count >= l.limit {
        return false
    }
    l.count++
    return true
}
```

这是全局限流，不区分用户。生产中通常按用户、IP、租户或 API key 限流。

---

## 五、实现最小熔断器

继续写：

```go
var ErrCircuitOpen = errors.New("circuit open")

type circuitBreaker struct {
    mu           sync.Mutex
    failures     int
    threshold    int
    openedUntil  time.Time
    openDuration time.Duration
}

func newCircuitBreaker(threshold int, openDuration time.Duration) *circuitBreaker {
    return &circuitBreaker{
        threshold:    threshold,
        openDuration: openDuration,
    }
}

func (c *circuitBreaker) beforeCall() error {
    c.mu.Lock()
    defer c.mu.Unlock()

    if time.Now().Before(c.openedUntil) {
        return ErrCircuitOpen
    }
    return nil
}

func (c *circuitBreaker) afterCall(err error) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if err == nil {
        c.failures = 0
        return
    }

    c.failures++
    if c.failures >= c.threshold {
        c.openedUntil = time.Now().Add(c.openDuration)
        c.failures = 0
    }
}
```

这个版本只有两个状态：

```text
closed：正常调用。
open：直接失败，不调用上游。
```

生产级熔断器还会有 half-open 状态，用少量请求试探上游是否恢复。本节先实现最小可观察版本。

---

## 六、实现上游调用和重试

继续写：

```go
func callUpstream(ctx context.Context, hc *http.Client) ([]byte, error) {
    var lastErr error

    for attempt := 0; attempt < 2; attempt++ {
        reqCtx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
        req, err := http.NewRequestWithContext(reqCtx, http.MethodGet, "http://127.0.0.1:9001/data", nil)
        if err != nil {
            cancel()
            return nil, err
        }

        resp, err := hc.Do(req)
        if err != nil {
            cancel()
            lastErr = err
        } else {
            body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
            resp.Body.Close()
            cancel()

            if readErr != nil {
                lastErr = readErr
            } else if resp.StatusCode >= 500 {
                lastErr = errors.New("upstream 5xx")
            } else if resp.StatusCode < 200 || resp.StatusCode >= 300 {
                return nil, errors.New("upstream non-2xx")
            } else {
                return body, nil
            }
        }

        if attempt == 0 {
            time.Sleep(100 * time.Millisecond)
        }
    }

    return nil, lastErr
}
```

这里固定最多尝试 2 次：

```text
第一次请求 + 1 次重试。
```

注意：

- 单次请求超时是 500ms。
- 响应体必须关闭。
- 只有 `5xx` 和网络错误会进入重试路径。
- 这里为了简化，没有对 `4xx` 重试。

---

## 七、组合 API 服务

继续写：

```go
func main() {
    l := newLimiter(5)
    cb := newCircuitBreaker(3, 5*time.Second)

    hc := &http.Client{
        Transport: &http.Transport{
            Proxy: http.ProxyFromEnvironment,
            DialContext: (&net.Dialer{
                Timeout:   200 * time.Millisecond,
                KeepAlive: 30 * time.Second,
            }).DialContext,
            MaxIdleConns:          100,
            MaxIdleConnsPerHost:   20,
            IdleConnTimeout:       60 * time.Second,
            ResponseHeaderTimeout: 700 * time.Millisecond,
        },
    }

    mux := http.NewServeMux()
    mux.HandleFunc("GET /proxy", func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        if !l.allow() {
            http.Error(w, "rate limited", http.StatusTooManyRequests)
            log.Printf("status=429 cost=%s", time.Since(start))
            return
        }

        if err := cb.beforeCall(); err != nil {
            http.Error(w, "circuit open", http.StatusServiceUnavailable)
            log.Printf("status=503 reason=circuit_open cost=%s", time.Since(start))
            return
        }

        body, err := callUpstream(r.Context(), hc)
        cb.afterCall(err)
        if err != nil {
            http.Error(w, "upstream failed", http.StatusBadGateway)
            log.Printf("status=502 err=%v cost=%s", err, time.Since(start))
            return
        }

        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        _, _ = w.Write(body)
        log.Printf("status=200 cost=%s", time.Since(start))
    })

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 3 * time.Second,
        ReadTimeout:       5 * time.Second,
        WriteTimeout:      5 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Println("api listening on :8080")
    log.Fatal(srv.ListenAndServe())
}
```

请求流程：

```text
入口限流 -> 熔断判断 -> 调上游 -> 根据结果更新熔断器 -> 返回响应
```

这个顺序很重要。限流放在最前面，可以尽早拒绝过载请求。熔断在调用前检查，避免明知上游失败还继续打。

---

## 八、正常运行验证

终端 1：

```powershell
go run ./cmd/upstream
```

终端 2：

```powershell
go run ./cmd/api
```

终端 3：

```powershell
curl.exe -i http://localhost:8080/proxy
```

预期：

```http
HTTP/1.1 200 OK
```

响应：

```json
{"message":"hello from upstream"}
```

---

## 九、验证限流

快速请求 10 次：

```powershell
1..10 | ForEach-Object {
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8080/proxy
}
```

你应该看到部分请求返回：

```text
429
```

这说明入口限流生效。

---

## 十、验证超时和重试

停止 upstream，用慢模式启动：

```powershell
go run ./cmd/upstream -slow 1s
```

API 单次上游超时是 500ms，所以请求会失败：

```powershell
curl.exe -i http://localhost:8080/proxy
```

预期：

```http
502 Bad Gateway
```

API 日志中会看到耗时大约超过 1 秒，因为它尝试了两次：

```text
第一次 500ms 超时。
等待 100ms。
第二次 500ms 超时。
```

这说明重试会增加总耗时，所以必须控制次数和总预算。

---

## 十一、验证熔断

继续保持 upstream 很慢，连续请求几次：

```powershell
curl.exe -i http://localhost:8080/proxy
curl.exe -i http://localhost:8080/proxy
curl.exe -i http://localhost:8080/proxy
curl.exe -i http://localhost:8080/proxy
```

前三次失败会让熔断器打开，后续请求直接返回：

```http
503 Service Unavailable
```

并且日志会出现：

```text
reason=circuit_open
```

此时 API 没有再调用上游，这是熔断的意义：保护自己，也给上游恢复空间。

等待 5 秒后，熔断器会允许再次尝试。

---

## 十二、常见问题

### 1. 超时越长越安全吗？

不是。

超时越长，资源占用越久。请求堆积后，goroutine、连接、内存都会被拖住。超时应该结合接口 SLA、上游能力和调用链总预算设置。

### 2. 重试一定能提高成功率吗？

只在临时故障时有帮助。

如果上游已经过载，重试会让请求量翻倍，故障更严重。所以重试必须有次数、退避和幂等性判断。

### 3. 熔断是不是会丢请求？

会主动拒绝一部分请求。

但这是为了避免所有请求都慢慢失败。快速失败通常比无限等待更容易恢复。

### 4. 限流应该放客户端还是服务端？

两边都可以。

客户端限流能减少对上游的压力；服务端限流能保护自己。网关、服务入口、下游客户端都可以有不同层次的限流。

---

## 十三、本节达标标准

学完本节后，你应该能够做到：

- 写出一个可模拟慢响应和 5xx 的上游服务。
- 为 API 服务配置上游 HTTP Client 的连接池和超时。
- 使用 `context.WithTimeout` 控制单次上游调用。
- 实现有限重试，并解释重试带来的耗时放大。
- 实现最小熔断器，理解 open/closed 状态。
- 实现固定窗口限流。
- 用 curl 观察 `200`、`429`、`502`、`503`。
- 解释超时、重试、熔断、限流分别解决什么问题。

