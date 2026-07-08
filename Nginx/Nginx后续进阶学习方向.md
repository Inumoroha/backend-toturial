# Nginx 后续进阶学习方向

这份文档用于回答一个问题：

```text
学完当前 Nginx + Go 后端路线后，后面还可以继续学什么？
```

前面 1 到 10 阶段已经覆盖了 Go 后端工程师使用 Nginx 的主线能力：

- Nginx 环境搭建。
- 配置结构、`server`、`location`、变量。
- 静态资源服务。
- Go API 反向代理。
- 代理头、真实 IP、超时、WebSocket。
- upstream 负载均衡。
- HTTPS 与安全基础。
- access log、error log、请求 ID。
- 502、503、504 排查。
- 限流、CORS、上传限制、代理缓存。
- worker、连接数、keepalive、压测。
- Go 服务 Nginx 网关项目落地。

到这里，你已经具备了在普通 Go 后端项目中使用 Nginx 的基础工程能力。

后续进阶不建议继续“横向背更多指令”，而是按照真实生产场景分方向深入。

---

## 一、进阶学习的总体思路

Nginx 后续进阶可以分成六条线：

```text
生产网关线。
性能优化线。
安全加固线。
高可用架构线。
可观测性线。
云原生与 API Gateway 线。
```

建议不要六条线同时学。

更好的方式是：

```text
先强化生产网关和日志排障。
再做一个更接近真实部署的项目。
项目中遇到性能、安全、高可用问题，再学习对应专题。
```

Nginx 学习最容易出现的问题是：

```text
配置片段看了很多，但不知道什么时候用。
线上问题遇到了，却不知道从哪一层开始查。
```

进阶阶段最重要的是：

```text
带着请求链路学习。
用日志和压测验证。
```

---

## 二、生产网关进阶

适合目标：

```text
能独立为 Go 后端服务设计一套稳定的 Nginx 网关配置。
```

### 需要继续学习

- 多域名网关配置。
- API 路由分组。
- 管理后台访问控制。
- 登录、验证码、上传接口的差异化策略。
- CORS 白名单治理。
- 请求体大小分层控制。
- 请求和响应 Header 规范。
- 灰度路由基础。
- 维护页与降级页。
- 配置拆分和发布回滚。

### 推荐练习

做一个多服务网关：

```text
app.example.com       -> 前端静态页面
api.example.com       -> Go API
admin.example.com     -> 管理后台
files.example.com     -> 文件下载服务
```

要求：

- 每个域名单独 server。
- API 和后台使用不同限流。
- 后台加 Basic Auth 或 IP 白名单。
- 上传接口单独放大 body size。
- 前端静态资源长缓存。
- HTML 不长缓存。
- 所有请求有统一 request_id。

### 学到什么程度算够

能做到：

- 不把所有 API 都塞进一个 location。
- 能按接口风险设计不同网关策略。
- 能解释每个 server 和 location 的职责。
- 能安全地发布和回滚 Nginx 配置。

---

## 三、性能优化进阶

适合目标：

```text
能判断性能瓶颈是在 Nginx、Go 服务、系统资源还是后端依赖。
```

### 需要继续学习

- worker 模型深入理解。
- 文件描述符限制。
- TCP backlog。
- keepalive 调优。
- upstream keepalive。
- gzip 与 CPU 开销。
- TLS 握手成本。
- HTTP/2。
- 静态文件高性能配置。
- 大文件下载优化。
- 压测方法论。
- Go pprof 与 Nginx 日志联合分析。

### 推荐练习

准备三个接口：

```text
/api/ping          快接口
/api/slow          慢接口
/api/cpu           CPU 密集接口
```

分别压测：

```text
Go 直连。
Nginx 代理。
Nginx + gzip。
Nginx + upstream keepalive。
HTTPS。
```

观察：

- QPS。
- 平均延迟。
- P95、P99。
- 错误率。
- CPU。
- 连接数。
- access log 中的 `rt`、`urt`、`uct`。

### 学到什么程度算够

能做到：

- 不靠猜测调参数。
- 能解释 `request_time` 和 `upstream_response_time`。
- 能判断慢在 Nginx 还是 Go。
- 能知道什么时候该优化业务，而不是调大 Nginx 超时。

---

## 四、安全加固进阶

适合目标：

```text
能让 Nginx 入口层具备基础安全防护能力。
```

### 需要继续学习

- TLS 版本与密码套件。
- HSTS。
- 安全响应头。
- 真实 IP 信任边界。
- `set_real_ip_from`。
- 防止 Host Header 攻击。
- 限制危险 HTTP 方法。
- Basic Auth 与后台保护。
- IP 黑白名单。
- 请求体限制。
- 防止敏感文件被访问。

### 推荐练习

给一个 Go API 做入口安全加固：

- 未知 Host 直接拒绝。
- HTTP 自动跳 HTTPS。
- 后台只允许内网访问。
- `/admin/` 加 Basic Auth。
- 禁止访问 `.git`、`.env` 等敏感路径。
- 给登录接口限流。
- 给上传接口限制大小。
- 配置安全响应头。

### 学到什么程度算够

能做到：

- 知道哪些安全能力适合 Nginx。
- 知道哪些安全逻辑必须放 Go。
- 不随便信任 `X-Forwarded-For`。
- 不把 Go 服务直接暴露公网。

---

## 五、高可用架构进阶

适合目标：

```text
能理解 Nginx 在多实例、多机器和云环境中的位置。
```

### 需要继续学习

- 多 Nginx 实例。
- 云负载均衡。
- Keepalived + VIP。
- upstream 健康检查思路。
- 被动健康检查与主动健康检查。
- 滚动发布。
- 蓝绿发布。
- 灰度发布。
- 故障切换。
- 多机日志收集。

### 推荐练习

设计一个两层入口架构：

```text
Cloud Load Balancer
  -> Nginx 1
  -> Nginx 2
      -> Go API 1
      -> Go API 2
      -> Go API 3
```

练习：

- 停掉一个 Go 实例。
- 停掉一个 Nginx 实例。
- 观察流量是否继续可用。
- 记录每层日志。
- 写出故障切换过程。

### 学到什么程度算够

能做到：

- 知道单台 Nginx 也是单点。
- 能解释 Nginx 和云负载均衡的关系。
- 能设计基本滚动发布流程。
- 能说明健康检查在高可用中的作用。

---

## 六、可观测性进阶

适合目标：

```text
能把 Nginx 纳入日志、指标和告警体系。
```

### 需要继续学习

- JSON 格式 access log。
- request_id 贯穿链路。
- 日志采集。
- Prometheus Nginx exporter。
- 状态码告警。
- 5xx 比例告警。
- P95、P99 延迟。
- upstream 响应时间监控。
- 证书过期告警。
- 磁盘日志轮转。

### 推荐练习

为 Nginx 建一个观测面板：

- 总请求数。
- 2xx、4xx、5xx 数量。
- 502、504 单独统计。
- 平均响应时间。
- P95 响应时间。
- upstream 响应时间。
- 每个 upstream 实例请求量。

同时配置告警：

- 5xx 比例超过阈值。
- 504 突然升高。
- 证书 15 天内过期。
- 日志磁盘空间不足。

### 学到什么程度算够

能做到：

- 线上出问题时先看指标，再看日志。
- 能用 request_id 串联 Nginx 和 Go 日志。
- 能区分客户端错误、网关错误和后端错误。
- 能建立基础告警，而不是等用户反馈。

---

## 七、云原生与 API Gateway 进阶

适合目标：

```text
理解 Nginx、Ingress、Kong、APISIX、Envoy、Traefik 的定位差异。
```

### 需要继续学习

- Kubernetes Ingress。
- Nginx Ingress Controller。
- Gateway API。
- Kong。
- APISIX。
- Envoy。
- Traefik。
- 服务发现。
- 动态路由。
- 插件体系。
- 认证鉴权插件。
- 熔断、限流、灰度。

### 推荐练习

把同一个 Go API 分别部署到：

```text
Docker Compose + Nginx
Kubernetes + Nginx Ingress
Kong 或 APISIX
```

对比：

- 路由配置方式。
- HTTPS 配置方式。
- 限流配置方式。
- 日志和指标。
- 服务发现。
- 灰度发布。

### 学到什么程度算够

能做到：

- 知道 Nginx 适合什么场景。
- 知道什么时候需要 API Gateway。
- 知道 Ingress 和普通 Nginx 配置的关系。
- 能根据团队规模和系统复杂度选择入口层方案。

---

## 八、推荐进阶项目

### 项目 1：多域名 API 网关

目标：

```text
一个 Nginx 管理多个域名和多个 Go 服务。
```

练习重点：

- server_name。
- upstream。
- 日志拆分。
- 限流策略。
- HTTPS。

### 项目 2：文件下载与上传网关

目标：

```text
支持大文件上传、静态文件下载、防盗链和缓存。
```

练习重点：

- `client_max_body_size`。
- 静态资源缓存。
- 权限控制。
- 大文件传输。

### 项目 3：高可用 Go API 入口层

目标：

```text
两个 Nginx + 三个 Go 实例 + 统一日志和故障演练。
```

练习重点：

- 多实例。
- 故障切换。
- 滚动发布。
- 502/504 排查。

### 项目 4：Kubernetes Ingress 迁移

目标：

```text
把当前 Nginx + Go 项目迁移到 Kubernetes Ingress。
```

练习重点：

- Ingress 规则。
- Service。
- TLS Secret。
- 配置注解。
- Ingress Controller 日志。

---

## 九、最终建议

学完当前 Nginx 路线后，不要急着背更多冷门指令。

建议按这个顺序继续：

1. 做一个多域名网关项目。
2. 强化日志、request_id 和 502/504 排查。
3. 学习生产安全加固。
4. 做压测和性能定位。
5. 再进入 Kubernetes Ingress 或 API Gateway。

Nginx 真正的进阶能力不是“知道更多配置”，而是：

```text
能设计入口层。
能解释请求链路。
能通过日志定位问题。
能在生产环境安全发布和回滚。
```

