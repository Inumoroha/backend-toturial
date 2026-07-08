# Nginx 系统教程总索引

这套教程面向想成为 Go 后端工程师的学习者。学习顺序从基础准备开始，到 Nginx 入门、反向代理、负载均衡、HTTPS、排障、网关能力、性能调优、生产部署，最后进入项目实战。

## 学习方式

建议你每学完一个文件，就完成里面的练习，不要只读配置。Nginx 的学习重点不是“记住所有指令”，而是能在真实服务里解释请求如何进入 Nginx、如何转发到 Go、如何记录日志、如何失败、如何恢复。

## 目录顺序

1. `01-preparation`：准备阶段，补齐 Linux、网络、Go HTTP 和本地实验环境。
2. `02-nginx-basics`：Nginx 基础，学习安装、启动、配置结构和命令。
3. `03-static-resources`：静态资源服务，学习 `root`、`alias`、缓存和 gzip。
4. `04-reverse-proxy-go`：反向代理 Go 服务，这是 Go 后端最核心的 Nginx 能力。
5. `05-load-balancing`：负载均衡，多 Go 实例、权重、失败处理。
6. `06-https-security`：HTTPS 与安全配置，理解 TLS 终止和常用安全头。
7. `07-logging-troubleshooting`：日志与排障，重点掌握 502、503、504。
8. `08-gateway-rate-limit-cache`：网关能力，限流、访问控制、CORS、缓存和上传限制。
9. `09-performance-tuning`：性能调优，理解连接、worker、压测与系统限制。
10. `10-deployment-production`：生产部署，systemd、Docker Compose、配置拆分和回滚。
11. `11-nginx-deep-dive`：深入理解 Nginx，学习 location、rewrite、变量和事件模型。
12. `12-project-practice`：项目实战，从小项目到最终综合部署。

## 文件导航

### `01-preparation`

- `01-linux-network-go-foundation.md`：Linux、网络和 Go HTTP 基础。
- `02-local-lab-setup.md`：搭建本地实验环境。
- `03-http-request-lifecycle.md`：一次 HTTP/HTTPS 请求从客户端到 Go 服务的完整链路。

### `02-nginx-basics`

- `01-install-start-config.md`：安装、启动、配置检查和 reload。
- `02-config-structure-location.md`：配置结构与 location 入门。
- `03-directive-context-include-process.md`：指令上下文、include 机制和 master/worker 进程模型。

### `03-static-resources`

- `01-static-site-root-alias-try-files.md`：静态站点、`root`、`alias`、`try_files`。
- `02-cache-gzip-and-errors.md`：缓存、gzip 和错误页。
- `03-permissions-mime-debugging.md`：权限、MIME 类型和静态资源排障。

### `04-reverse-proxy-go`

- `01-go-api-and-proxy-pass.md`：Go API 与 `proxy_pass`。
- `02-headers-timeouts-websocket.md`：代理头、超时、请求体和 WebSocket。
- `03-real-ip-trust-boundary.md`：真实 IP、代理头伪造和信任边界。

### `05-load-balancing`

- `01-upstream-load-balancing.md`：`upstream` 与多 Go 实例负载均衡。
- `02-failure-and-session-strategy.md`：失败处理、重试和无状态会话。
- `03-idempotency-graceful-restart.md`：幂等、优雅重启和发布风险。

### `06-https-security`

- `01-https-certificates-and-redirect.md`：HTTPS、证书、TLS 终止和跳转。
- `02-security-hardening.md`：安全响应头、IP 控制和 Basic Auth。
- `03-certbot-renewal-tls-checklist.md`：Certbot 续期和 TLS 上线检查。

### `07-logging-troubleshooting`

- `01-access-error-logs.md`：access log、error log 和 upstream 耗时。
- `02-502-503-504-troubleshooting.md`：502、503、504 系统排查。
- `03-request-id-log-correlation.md`：请求 ID 与 Nginx/Go 日志关联。

### `08-gateway-rate-limit-cache`

- `01-rate-limit-and-access-control.md`：限流、连接限制、IP 控制和 Basic Auth。
- `02-cors-cache-upload.md`：CORS、代理缓存和上传限制。
- `03-production-gateway-patterns.md`：按接口类型设计生产网关策略。

### `09-performance-tuning`

- `01-performance-model-and-system-limits.md`：worker、连接数、keepalive 和系统限制。
- `02-benchmark-and-tuning.md`：压测、观察和调参方法。
- `03-os-go-pprof-and-bottleneck.md`：系统瓶颈、Go pprof 和定位方法。

### `10-deployment-production`

- `01-systemd-deploy-go-nginx.md`：systemd 管理 Go 服务，Nginx 对外代理。
- `02-docker-compose-deploy.md`：Docker Compose 部署 Nginx + Go。
- `03-config-layout-release-rollback.md`：配置拆分、发布流程和回滚。

### `11-nginx-deep-dive`

- `01-location-rewrite-variables.md`：location、rewrite 和变量。
- `02-event-model-and-boundaries.md`：事件模型和 Nginx/Go 职责边界。
- `03-request-processing-phases.md`：Nginx 请求处理阶段。

### `12-project-practice`

- `01-project-go-api-proxy.md`：Go API 反向代理项目。
- `02-project-load-balancing.md`：三实例 Go API 负载均衡项目。
- `03-project-https-gateway.md`：HTTPS、限流与安全网关项目。
- `04-project-final-capstone.md`：最终综合部署项目。
- `05-final-project-complete-go-source.md`：最终项目完整 Go 源码与运行说明。

## 推荐节奏

- 第 1 周：完成 `01`、`02`。
- 第 2 周：完成 `03`、`04`。
- 第 3 周：完成 `05`、`06`。
- 第 4 周：完成 `07`、`08`。
- 第 5 周：完成 `09`、`10`。
- 第 6 周：完成 `11`、`12`。

每个阶段都可以反复练。Nginx 真正的熟练感来自配置、请求、日志、错误现象之间的来回验证。
