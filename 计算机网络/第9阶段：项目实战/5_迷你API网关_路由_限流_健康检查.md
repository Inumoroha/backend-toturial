# 5. 迷你 API 网关：路由、限流、健康检查

本节目标：在反向代理基础上设计迷你 API 网关，支持路径路由、限流和健康检查。

---

## 一、路由配置

```yaml
routes:
  - path_prefix: /users
    upstreams:
      - http://127.0.0.1:8081
    timeout_ms: 1000
    rate_limit_per_second: 100
```

---

## 二、路径匹配

```go
func matchRoute(path string, routes []Route) *Route {
    var matched *Route
    for i := range routes {
        r := &routes[i]
        if strings.HasPrefix(path, r.PathPrefix) {
            if matched == nil || len(r.PathPrefix) > len(matched.PathPrefix) {
                matched = r
            }
        }
    }
    return matched
}
```

---

## 三、限流

可以使用令牌桶：

```go
limiter := rate.NewLimiter(rate.Limit(100), 200)

if !limiter.Allow() {
    http.Error(w, "too many requests", http.StatusTooManyRequests)
    return
}
```

---

## 四、健康检查

定期请求上游：

```text
GET /health
```

失败次数超过阈值后临时摘除，恢复后再加入。

---

## 补充落地步骤：网关先做最小主链路

第一版只做：

```text
读取静态路由配置。
按 path prefix 匹配。
转发到对应 upstream。
记录访问日志。
```

第二版再加：

```text
Token 鉴权。
固定窗口限流。
健康检查。
错误响应统一格式。
```

这样拆的好处是：每次只引入一个变量，出问题时容易定位。

---

## 补充验收：限流要能被证明

限流不是写了代码就算完成。要用命令验证：

```bash
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/users/1
done
```

你应该能看到：

```text
200
200
429
```

日志里也要有：

```text
reason=rate_limited
```

这样才能证明请求是在网关层被拒绝，而不是业务服务自己失败。

---

## 补充配置：路由规则要避免歧义

如果同时有：

```text
/users
/users/admin
```

匹配时要考虑最长前缀优先，否则 `/users/admin` 可能被普通 `/users` 路由提前匹配。

建议路由配置里写清楚：

```text
匹配方式：prefix / exact。
优先级：priority 或最长前缀。
是否去掉前缀再转发。
超时时间。
限流阈值。
```

网关项目的难点不是转发，而是规则越来越多以后仍然可预测。

---

## 补充验收：健康检查不要经过鉴权

网关自身健康检查通常应允许平台直接访问：

```bash
curl -i http://127.0.0.1:8080/healthz
```

预期：

```text
不需要 Authorization。
返回 200 或 204。
不会被限流误伤。
不会转发到业务上游。
```

如果健康检查也被鉴权或限流拦截，负载均衡可能误判实例不健康。

---

## 补充日志：健康检查通常单独降噪

健康检查频率很高，日志可以单独标记：

```text
path=/healthz type=healthcheck
```

避免它淹没真实业务请求日志。

---

## 五、常见问题

### 1. 路由匹配为什么用最长前缀？

避免 `/users` 抢先匹配 `/users/admin` 这类更具体的路由。

### 2. 健康检查是否一次失败就摘除？

通常不建议。应设置失败阈值，避免短暂抖动导致频繁摘除。

### 3. 限流只在网关做够吗？

不一定。网关保护入口，服务内还可以按用户、资源、业务动作做更细限流。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 设计网关配置。
- 实现路径前缀匹配。
- 使用限流返回 429。
- 设计上游健康检查。
- 区分网关 404、429、502、504。
