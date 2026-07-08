# 教程质量覆盖检查表

这份文件用于检查整套 Nginx 教程是否真正覆盖路线图要求，而不是只列标题。

## 一、覆盖标准

每个阶段至少应包含：

- 学习目标：说明本阶段解决什么问题。
- 核心概念：解释为什么要学，不只给配置。
- 可运行配置：给出可以复制实验的 Nginx 或 Go 示例。
- 验证命令：使用 `curl`、`nginx -t`、日志或系统命令验证结果。
- 常见错误：说明出错时应该查什么。
- 阶段验收：学习者知道自己是否学会。

## 二、路线图覆盖情况

| 路线图模块 | 已覆盖文件 | 需要重点掌握 |
| --- | --- | --- |
| Linux、网络、Go 基础 | `01-preparation/*` | 命令、端口、HTTP、Go HTTP 服务 |
| Nginx 入门 | `02-nginx-basics/*` | 安装、启动、配置结构、include、上下文 |
| 静态资源 | `03-static-resources/*` | `root`、`alias`、`try_files`、缓存、权限 |
| 反向代理 Go | `04-reverse-proxy-go/*` | `proxy_pass`、代理头、真实 IP、安全边界 |
| 负载均衡 | `05-load-balancing/*` | `upstream`、失败处理、幂等、无状态 |
| HTTPS 与安全 | `06-https-security/*` | 证书、TLS 终止、跳转、续期、安全头 |
| 日志与排障 | `07-logging-troubleshooting/*` | access/error log、请求 ID、502/503/504 |
| 限流、缓存、网关 | `08-gateway-rate-limit-cache/*` | 限流、CORS、缓存、上传限制 |
| 性能调优 | `09-performance-tuning/*` | worker、连接数、压测、pprof、系统限制 |
| 部署实践 | `10-deployment-production/*` | systemd、Docker Compose、配置拆分、回滚 |
| 深入理解 | `11-nginx-deep-dive/*` | location、rewrite、变量、请求处理阶段 |
| 项目实战 | `12-project-practice/*` | 反向代理、负载均衡、HTTPS、综合部署 |

## 三、学习者自测问题

学完后你应该能回答：

1. 一个 HTTPS 请求从浏览器到 Go handler，中间经过了哪些步骤？
2. `server_name` 和 `location` 分别在什么时候参与匹配？
3. `root` 与 `alias` 的路径拼接差异是什么？
4. `proxy_pass` 后面带 `/` 与不带 `/` 的区别是什么？
5. Go 服务为什么不能无条件相信 `X-Forwarded-For`？
6. Nginx 返回 502 时第一步看什么？
7. Nginx 返回 504 时应该优先改超时还是查 Go 服务？
8. 多实例部署时为什么业务服务要尽量无状态？
9. Nginx 限流和 Go 业务限流的边界在哪里？
10. systemd 部署和 Docker Compose 部署分别适合什么场景？

## 四、最低合格线

只读完文档不算完成。最低合格线是：

- 至少亲手部署 1 个静态站点。
- 至少亲手代理 1 个 Go API。
- 至少亲手配置 1 次 upstream 三实例负载均衡。
- 至少亲手复现并排查 502、504。
- 至少亲手完成最终综合项目。

