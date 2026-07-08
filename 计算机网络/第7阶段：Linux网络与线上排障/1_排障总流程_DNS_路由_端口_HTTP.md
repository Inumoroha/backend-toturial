# 1. 排障总流程：DNS、路由、端口、HTTP

本节目标：建立一套稳定的网络排障顺序，避免线上问题一上来就猜代码。

---

## 一、推荐排障顺序

```text
1. 域名能不能解析？
2. 目标 IP 有没有路由？
3. 目标主机是否可达？
4. 目标端口是否开放？
5. TCP 是否建立成功？
6. TLS 是否握手成功？
7. HTTP 是否返回状态码？
8. 服务端是否收到请求？
9. 应用内部是否慢？
```

---

## 二、命令模板

DNS：

```bash
dig api.example.com
```

路由：

```bash
ip route get 目标IP
```

端口：

```bash
nc -vz 目标IP 端口
```

HTTP：

```bash
curl -v https://api.example.com
```

抓包：

```bash
sudo tcpdump -i any host 目标IP -nn
```

---

## 三、判断思路

如果 DNS 失败，服务端不会有日志。

如果 TCP 失败，HTTP 还没开始。

如果 TLS 失败，HTTP handler 可能不会执行。

如果 HTTP 返回 5xx，说明请求通常已经到达网关或服务。

---

## 补充模板：一次排障先记录事实

不要一开始就猜“是不是代码 bug”。先写一份最小排障单：

```text
故障现象：
请求方机器/容器：
目标域名：
目标端口：
协议：http / https / tcp / udp
失败比例：全部失败 / 偶发失败 / 某个机房失败
开始时间：
最近变更：
客户端错误：
服务端是否有日志：
```

示例：

```text
故障现象：order-service 调 user-service 超时。
请求方：prod-order-03 容器。
目标域名：user.internal。
目标端口：443。
协议：https。
失败比例：只在线上失败，测试环境正常。
开始时间：10:05。
最近变更：10:00 调整过安全组。
客户端错误：context deadline exceeded。
服务端是否有日志：user-service 没有请求日志。
```

这份记录会自然把你引向 DNS、路由、端口、安全组，而不是一头扎进业务代码。

---

## 补充演练：从客户端到服务端逐层证明

按顺序执行：

```bash
dig user.internal
```

证明：

```text
域名是否能解析。
解析到哪个 IP。
TTL 是多少。
```

继续：

```bash
ip route get 解析到的IP
```

证明：

```text
本机会从哪个网卡出去。
源 IP 是什么。
下一跳是谁。
```

继续：

```bash
curl -v --connect-timeout 3 https://user.internal/healthz
```

证明：

```text
TCP 是否连上。
TLS 是否成功。
HTTP 是否返回状态码。
```

如果仍然不清楚，抓包：

```bash
sudo tcpdump -i any -nn host 解析到的IP and port 443
```

判断：

```text
没有包：请求没发出或过滤条件错。
只有 SYN：包发出但没有回应。
有 SYN-ACK：TCP 通了，继续看 TLS/HTTP。
有 HTTP 响应：网络层基本通，继续看业务状态码和日志。
```

这样排障结论会有证据，而不是“感觉像网络问题”。

---

## 补充输出：最后要给出可复核结论

排障结束时，建议按这个格式输出：

```text
结论：
证据 1：
证据 2：
排除项：
根因：
修复动作：
验证方式：
后续预防：
```

示例：

```text
结论：请求没有到达 user-service，失败发生在 TCP 建连阶段。
证据 1：dig 解析 user.internal 到 10.20.1.15。
证据 2：tcpdump 只看到客户端 SYN 重传，没有 SYN-ACK。
排除项：DNS 正常，客户端路由正常。
根因：安全组未放行 order-service 所在网段。
修复动作：放行 10.30.2.0/24 到 443。
验证方式：curl -v 返回 HTTP 200，服务端 access log 出现 request id。
后续预防：安全组变更加入发布检查清单。
```

能写出这种结论，说明你不是在“试错”，而是在做工程排障。

---

## 四、常见问题

### 1. 服务端没有日志说明什么？

请求可能根本没到服务端，也可能被网关拦截。先看 DNS、TCP、网关日志和抓包。

### 2. ping 通但服务不可用怎么办？

继续测 TCP 端口和 HTTP，不要停在 ping。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 按层次排查网络问题。
- 用命令区分 DNS、路由、端口、HTTP 问题。
- 解释为什么服务端没日志时不应只看业务代码。
