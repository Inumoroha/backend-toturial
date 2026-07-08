# 0. 反向代理 Go 后端服务总览

本阶段目标：把 Nginx 放到 Go 服务前面，让 Nginx 对外接收请求，再转发给 Go API。

这是 Go 后端工程师学习 Nginx 的核心阶段。

```text
Client
  |
  | HTTP / HTTPS
  v
Nginx
  |
  | HTTP
  v
Go API
```

---

## 一、本阶段你要掌握什么

- 准备 Go HTTP 服务。
- `proxy_pass` 基础。
- `proxy_pass` 路径规则。
- `Host`、`X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-Proto`。
- 真实 IP 的信任边界。
- 连接超时、发送超时、读取超时。
- 请求体大小限制。
- WebSocket 代理。

---

## 二、本阶段文档顺序

建议按下面顺序学习：

1. `1_准备一个用于Nginx代理的Go服务.md`
2. `2_proxy_pass基础与路径规则.md`
3. `3_代理头真实IP与协议传递.md`
4. `4_超时请求体大小与504排查.md`
5. `5_WebSocket代理配置.md`

---

## 三、为什么不直接暴露 Go 服务

Go 服务当然可以直接监听公网端口，但生产中通常不推荐。

Nginx 放在前面可以：

- 统一 HTTPS。
- 隐藏 Go 服务真实端口。
- 做负载均衡。
- 记录统一访问日志。
- 设置上传大小。
- 做基础限流。
- 屏蔽一部分异常请求。

Go 服务更专注业务逻辑。

---

## 四、本阶段最小闭环

```text
启动 Go 服务 :8080
-> curl 直连 Go 成功
-> Nginx 监听 :80
-> proxy_pass 到 Go
-> curl 访问 Nginx 成功
-> Go 打印代理头
-> Nginx access.log 有记录
```

---

## 五、本阶段验收

完成后你应该能：

1. 写一个 Go `/api/ping` 接口。
2. 让 Nginx 转发到 Go。
3. 解释 `proxy_pass` 后面带 `/` 和不带 `/` 的区别。
4. 在 Go 中看到 `X-Real-IP` 和 `X-Forwarded-For`。
5. 制造一个 504 并通过日志解释原因。

