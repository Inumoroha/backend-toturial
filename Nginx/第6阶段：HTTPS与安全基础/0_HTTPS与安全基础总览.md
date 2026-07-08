# 0. HTTPS 与安全基础总览

本阶段目标：让 Nginx 对外提供 HTTPS，并理解 TLS 终止、证书、私钥、HTTP 跳转 HTTPS 和基础安全配置。

生产环境中常见链路：

```text
Client --HTTPS--> Nginx --HTTP--> Go
```

Nginx 负责 HTTPS，Go 服务继续跑 HTTP。这叫 TLS 终止。

---

## 一、本阶段你要掌握什么

- 自签名证书。
- Let's Encrypt 证书。
- Certbot。
- `listen 443 ssl`。
- `ssl_certificate`。
- `ssl_certificate_key`。
- HTTP 自动跳转 HTTPS。
- `X-Forwarded-Proto`。
- 安全响应头。
- Basic Auth。
- IP 访问控制。
- 证书续期检查。

---

## 二、本阶段文档顺序

建议按下面顺序学习：

1. `1_自签名证书与HTTPS最小配置.md`
2. `2_HTTP跳转HTTPS与X_Forwarded_Proto.md`
3. `3_Certbot证书续期与安全响应头.md`

---

## 三、为什么 HTTPS 通常放在 Nginx

原因：

- Nginx 对 TLS 支持成熟。
- 证书集中管理。
- 多个 Go 服务不用各自处理 HTTPS。
- 负载均衡和反向代理更方便。
- Go 服务可以只暴露内网端口。

---

## 四、本阶段验收

完成后你应该能：

1. 生成自签名证书。
2. 配置 `443 ssl`。
3. 使用 `curl -k` 访问 HTTPS。
4. 配置 HTTP 301 跳 HTTPS。
5. 让 Go 服务看到 `X-Forwarded-Proto: https`。
6. 说清楚证书续期为什么重要。

