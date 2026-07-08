# 0. DNS、HTTP、HTTPS 与浏览器访问流程总览

本阶段目标：完整理解一次浏览器访问 HTTPS 网站的过程，并掌握 DNS、HTTP、TLS、状态码、Header、Cookie、缓存、HTTP/2、HTTP/3 在后端开发中的作用。

---

## 一、本阶段学习顺序

```text
1_DNS解析流程与dig实验
2_HTTP请求响应报文与curl观察
3_HTTP方法状态码Header与后端语义
4_Cookie_Session_Token与缓存控制
5_HTTPS_TLS证书CA与SNI
6_HTTP1_HTTP2_HTTP3与QUIC
7_Go_HTTP_Client超时连接池与请求取消
8_浏览器访问HTTPS网站完整复盘
```

---

## 二、本阶段达标标准

学完本阶段后，你应该能够做到：

- 用 `dig` 查询 DNS。
- 用 `curl -v` 观察 HTTP/TLS 过程。
- 解释常见 HTTP 状态码。
- 区分 Cookie、Session、Token。
- 解释 HTTPS 如何加密、防篡改、认证身份。
- 解释 HTTP/1.1、HTTP/2、HTTP/3 的核心差异。
- 写出可靠的 Go HTTP Client 配置。
- 讲清浏览器访问 `https://example.com` 的完整流程。

