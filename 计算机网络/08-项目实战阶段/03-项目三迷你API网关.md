# 03-项目三：迷你 API 网关

## 项目目标

在反向代理基础上实现一个迷你 API 网关。它不追求复杂功能，而是帮助你理解真实后端入口层常见能力：路由、鉴权、限流、超时、负载均衡、健康检查和观测。

## 一、功能要求

基础功能：

- 根据路径转发到不同服务。
- 支持静态服务配置。
- 统一请求日志。
- 统一超时控制。
- 简单限流。
- 健康检查。

进阶功能：

- 动态加载配置。
- JWT 鉴权。
- 熔断。
- 灰度发布。
- Prometheus 指标。

## 二、配置设计

使用 YAML 或 JSON：

```yaml
routes:
  - path_prefix: /users
    upstreams:
      - http://127.0.0.1:8081
      - http://127.0.0.1:8082
    timeout_ms: 1000
    rate_limit_per_second: 100

  - path_prefix: /orders
    upstreams:
      - http://127.0.0.1:8091
    timeout_ms: 1500
    rate_limit_per_second: 50
```

## 三、核心模块

```text
config       读取配置
router       根据 path 匹配 route
balancer     选择 upstream
proxy        转发请求
middleware   日志、限流、鉴权
health       健康检查
metrics      指标
```

## 四、请求流程

```text
接收请求
  -> 生成 request id
  -> 记录开始时间
  -> 匹配路由
  -> 限流
  -> 鉴权
  -> 选择上游
  -> 带超时转发
  -> 记录状态码和耗时
```

## 五、路由匹配

简单实现可以用最长前缀匹配：

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

## 六、限流

每个 route 一个 limiter：

```go
type RouteRuntime struct {
    Route   Route
    Limiter *rate.Limiter
}
```

请求进入时：

```go
if !runtime.Limiter.Allow() {
    http.Error(w, "too many requests", http.StatusTooManyRequests)
    return
}
```

## 七、健康检查

网关应定期检查上游：

```text
GET /health
```

如果上游失败次数超过阈值，临时摘除。恢复后再加入。

注意：健康检查不能太频繁，也不能因为一次失败就立刻摘除。

## 八、观测能力

至少记录：

- request id。
- method。
- path。
- route。
- upstream。
- status。
- cost。
- error。

日志示例：

```text
request_id=abc method=GET path=/users/1 upstream=http://127.0.0.1:8081 status=200 cost=12ms
```

## 九、验收标准

必须完成：

- `/users` 和 `/orders` 能转发到不同后端。
- 路由未匹配时返回 404。
- 超过限流时返回 429。
- 上游不可用时返回 502。
- 上游慢响应时返回 504。
- 日志能看到请求转发详情。

加分项：

- 配置文件热加载。
- JWT 鉴权。
- Prometheus 指标。
- 熔断。

