# 5. HTTP 反向代理：负载均衡、超时、日志

本节目标：在反向代理中加入多个上游、轮询负载均衡、Transport 超时和请求日志。

---

## 一、轮询选择上游

```go
type Balancer struct {
    mu      sync.Mutex
    targets []*url.URL
    next    int
}

func (b *Balancer) Pick() *url.URL {
    b.mu.Lock()
    defer b.mu.Unlock()
    t := b.targets[b.next%len(b.targets)]
    b.next++
    return t
}
```

---

## 二、自定义 Director

```go
proxy := &httputil.ReverseProxy{
    Director: func(r *http.Request) {
        target := balancer.Pick()
        r.URL.Scheme = target.Scheme
        r.URL.Host = target.Host
        r.Host = target.Host
    },
}
```

---

## 三、Transport 超时

```go
proxy.Transport = &http.Transport{
    MaxIdleConns:          100,
    MaxIdleConnsPerHost:   20,
    IdleConnTimeout:       90 * time.Second,
    ResponseHeaderTimeout: 2 * time.Second,
}
```

---

## 四、日志中间件

```go
func logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("method=%s path=%s cost=%s", r.Method, r.URL.Path, time.Since(start))
    })
}
```

---

## 补充验收：负载均衡不是只看能不能访问

启动两个上游：

```text
backend-a :9001
backend-b :9002
```

连续请求代理：

```bash
for i in $(seq 1 10); do curl -s http://127.0.0.1:8080/; done
```

验收：

```text
两个上游都收到请求。
日志能显示选中的 upstream。
停止一个上游后，错误能被记录。
超时上游不会让代理永久卡住。
```

如果只是“请求能返回 200”，还不能证明负载均衡逻辑正确。

---

## 补充日志字段：代理项目必须有 upstream 信息

建议日志：

```text
method=GET path=/api upstream=http://127.0.0.1:9001 status=200 cost=12ms
```

没有 upstream 字段时，线上排查会很痛苦：你不知道慢请求到底被转发到了哪个后端。

---

## 补充排障：轮询为什么看起来不均匀

短时间请求量很少时，轮询可能因为连接复用、浏览器缓存、测试命令间隔等因素看起来不均匀。

建议用脚本连续请求：

```bash
for i in $(seq 1 20); do curl -s http://127.0.0.1:8080/; done
```

并同时看两个后端日志。

如果仍然只打到一个后端，检查：

```text
upstream 列表是否只有一个。
轮询计数是否并发安全。
Director 是否每次都重新选择上游。
错误上游是否被提前过滤。
```

---

## 补充验收：超时必须能复现

让一个后端故意 sleep：

```text
backend-a sleep 5s
backend-b 正常返回
```

代理设置：

```text
ResponseHeaderTimeout = 2s
```

验收：

```text
打到 backend-a 的请求应超时并记录错误。
打到 backend-b 的请求应正常返回。
日志里能看到 upstream 和 cost。
```

如果超时无法复现，说明你的代理还没有真正具备排障价值。

---

## 五、常见问题

### 1. 轮询负载均衡是否总是公平？

不一定。请求耗时不同、实例性能不同、长连接复用都会影响实际分布。

### 2. ResponseHeaderTimeout 控制什么？

控制代理向上游发出请求后等待响应头的时间。

### 3. 日志为什么要记录耗时？

耗时能帮助区分网关慢、上游慢和客户端慢，是排障核心字段。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 实现轮询选择上游。
- 配置代理 Transport。
- 记录请求方法、路径、耗时。
- 解释上游慢响应为什么会导致 504。
