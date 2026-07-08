# 02 深入理解：事件模型与职责边界

## 本节目标

理解 Nginx 为什么适合处理高并发连接，以及它和 Go 服务之间的职责边界。

## 一、Nginx 为什么适合做入口层

Nginx 使用事件驱动、非阻塞 I/O、多 worker 进程模型，适合处理大量连接和 I/O 转发。

它擅长：

- 接入 HTTP/HTTPS 流量。
- 静态资源服务。
- 反向代理。
- 负载均衡。
- 基础限流。
- TLS 终止。
- 日志记录。

## 二、Go 服务擅长什么

Go 服务擅长：

- 业务规则。
- 数据库读写。
- 用户认证授权。
- 复杂限流。
- 消息队列。
- 分布式事务或幂等控制。
- 领域模型和业务状态管理。

不要把复杂业务逻辑硬塞进 Nginx 配置。

## 三、职责边界建议

适合放 Nginx：

- 域名和路径转发。
- HTTPS。
- 静态文件。
- 单 IP 粗限流。
- 上传大小限制。
- 基础安全头。
- 统一访问日志。

适合放 Go：

- 用户登录态。
- RBAC 权限。
- 按用户或租户限流。
- 订单创建。
- 支付回调验签。
- 业务缓存策略。
- 接口幂等。

## 四、Nginx 与 API Gateway

Nginx 可以承担一部分网关职责，但完整 API Gateway 往往还需要：

- 服务发现。
- 动态路由。
- 插件系统。
- 认证鉴权。
- 指标监控。
- 熔断降级。
- 灰度发布。

常见网关：

- Kong。
- Traefik。
- Envoy。
- APISIX。
- 云厂商 API Gateway。

Go 后端工程师需要知道：Nginx 是很好的入口层，但不是所有网关问题的唯一答案。

## 五、常见架构形态

### 小型单机

```text
Client -> Nginx -> Go API -> Database
```

### 多实例

```text
Client -> Nginx -> Go API 1
                -> Go API 2
                -> Go API 3
```

### 容器或云环境

```text
Client -> Load Balancer -> Nginx/Ingress -> Go Service -> Database
```

## 六、本节练习

1. 列出你的项目中哪些逻辑应该放 Nginx。
2. 列出哪些逻辑必须放 Go。
3. 画出一个 Go + Nginx + Redis + MySQL 的请求链路。
4. 思考用户登录态应该如何在多实例间共享。
5. 对比 Nginx 与 API Gateway 的职责差异。

## 七、你应该掌握

学完本节，你应该能从架构角度判断：

- Nginx 适合解决什么问题。
- Go 服务应该保留哪些业务职责。
- 什么时候需要更完整的 API Gateway。

