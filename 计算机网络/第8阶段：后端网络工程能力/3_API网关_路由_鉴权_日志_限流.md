# 3. API 网关：路由、鉴权、日志、限流

本节目标：理解 API 网关常见职责，并能设计一个迷你网关的基础模块。

---

## 一、API 网关做什么

网关位于客户端和后端服务之间：

```text
客户端 -> API 网关 -> 用户服务/订单服务/支付服务
```

常见职责：

- 路由转发。
- 鉴权。
- 限流。
- 请求日志。
- 超时控制。
- 负载均衡。
- 灰度发布。

---

## 二、核心请求流程

```text
接收请求。
生成 request id。
匹配路由。
鉴权。
限流。
选择上游。
转发请求。
记录状态码和耗时。
```

---

## 三、路由配置示例

```yaml
routes:
  - path_prefix: /users
    upstreams:
      - http://127.0.0.1:8081
    timeout_ms: 1000
    rate_limit_per_second: 100
```

---

## 四、Go 实现思路

模块拆分：

```text
config    读取配置
router    匹配 path
balancer  选择 upstream
proxy     转发请求
middleware 日志、鉴权、限流
```

---

## 补充设计：网关请求日志应该包含什么

网关是入口层，日志至少要包含：

```text
request_id
method
path
status
duration_ms
client_ip
upstream
upstream_status
error
```

示例：

```text
request_id=req-001 method=GET path=/users/1 status=200 duration_ms=12 upstream=user-service upstream_status=200
```

当用户说“接口慢”时，你可以通过网关日志判断：

```text
请求有没有进网关。
有没有通过鉴权。
有没有被限流。
转发到哪个上游。
上游耗时是多少。
最终状态码是谁生成的。
```

---

## 补充代码：最小路由匹配思路

```go
type Route struct {
    Prefix   string
    Upstream string
}

func match(routes []Route, path string) (Route, bool) {
    for _, rt := range routes {
        if strings.HasPrefix(path, rt.Prefix) {
            return rt, true
        }
    }
    return Route{}, false
}
```

示例配置：

```go
routes := []Route{
    {Prefix: "/users/", Upstream: "http://127.0.0.1:8081"},
    {Prefix: "/orders/", Upstream: "http://127.0.0.1:8082"},
}
```

注意路径前缀最好写成 `/users/` 而不是 `/users`，否则 `/users2` 也可能被误匹配。

第 9 阶段的迷你 API 网关项目会把这个思路完整落成可运行代码。

---

## 补充边界：哪些逻辑不应该塞进网关

网关适合做横切能力：

```text
认证。
限流。
路由。
灰度。
日志。
基础协议转换。
```

不适合承载大量业务规则：

```text
订单金额计算。
库存扣减。
用户等级权益判断。
复杂审批流。
```

原因：

```text
网关会变成所有业务的耦合中心。
发布风险变大。
团队边界不清楚。
排障时很难判断业务逻辑到底在哪一层。
```

一句话：网关要强，但不要变成业务服务。

---

## 补充现场判断：网关问题还是业务服务问题

排障时先看网关日志：

```text
没有网关日志：请求没到入口，查 DNS、LB、WAF。
网关返回 401/403：查认证和权限。
网关返回 429：查限流规则和调用方流量。
网关返回 502/504：查 upstream、超时和业务服务健康。
网关 200 但业务失败：继续看业务响应体和服务日志。
```

这样能避免所有问题都丢给业务服务，也能避免网关层悄悄吞掉真实错误。

---

## 五、常见问题

### 1. 网关是不是越多功能越好？

不是。网关太重会成为瓶颈和复杂度中心。

### 2. 限流放网关还是服务内？

可以都放。网关做入口保护，服务内做业务维度保护。

### 3. request id 有什么用？

用于串联网关、服务、下游日志。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 解释 API 网关职责。
- 设计基础网关请求流程。
- 说明路由、鉴权、日志、限流模块的作用。
