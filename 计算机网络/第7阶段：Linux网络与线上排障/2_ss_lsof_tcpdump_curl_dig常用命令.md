# 2. ss、lsof、tcpdump、curl、dig 常用命令

本节目标：掌握后端排障最常用的几个网络命令，知道每个命令能证明什么。

---

## 一、dig

```bash
dig example.com
dig +short example.com
dig @8.8.8.8 example.com
```

用于证明 DNS 解析结果。

---

## 二、curl

```bash
curl -v http://127.0.0.1:8080
curl -i http://127.0.0.1:8080/health
curl --max-time 3 http://127.0.0.1:8080
```

用于观察 TCP、TLS、HTTP 状态。

---

## 三、ss

监听端口：

```bash
ss -lntp
```

所有 TCP 连接：

```bash
ss -antp
```

按状态：

```bash
ss -ant state time-wait
ss -ant state close-wait
```

---

## 四、lsof

查看端口进程：

```bash
lsof -i :8080
```

查看进程打开文件：

```bash
lsof -p 进程ID
```

---

## 五、tcpdump

抓端口：

```bash
sudo tcpdump -i any port 8080 -nn
```

抓主机：

```bash
sudo tcpdump -i any host 目标IP -nn
```

保存文件：

```bash
sudo tcpdump -i any port 8080 -w traffic.pcap
```

---

## 补充速查：命令能证明什么

| 命令 | 能证明 | 不能证明 |
| --- | --- | --- |
| `dig` | DNS 返回了什么 | 服务端是否可访问 |
| `curl -v` | DNS/TCP/TLS/HTTP 阶段现象 | 服务端内部为什么慢 |
| `ss -lntp` | 本机端口是否监听 | 防火墙是否放行 |
| `lsof -i :8080` | 端口属于哪个进程 | 请求是否能从远端到达 |
| `tcpdump` | 本机网卡是否看到包 | 对端业务是否处理成功 |

排障时最常见的错误是：用一个命令试图证明所有事情。

比如 `ping` 通只能说明 ICMP 有回应，不能说明 TCP 端口开放，更不能说明 HTTP 服务正常。

---

## 补充组合：一分钟判断服务是否真的在监听

先看监听：

```bash
ss -lntp | grep ':8080'
```

看进程：

```bash
lsof -i :8080
```

本机访问：

```bash
curl -i http://127.0.0.1:8080/healthz
```

用本机 IP 访问：

```bash
curl -i http://本机IP:8080/healthz
```

这四步能区分：

```text
服务没启动。
服务启动但监听 127.0.0.1。
服务监听 0.0.0.0 但防火墙阻断。
服务可访问但 HTTP 路由或业务错误。
```

如果 `127.0.0.1` 成功、本机 IP 失败，优先怀疑监听地址或防火墙。

---

## 补充组合：证明请求有没有到达本机

在服务端机器执行抓包：

```bash
sudo tcpdump -i any -nn 'tcp port 8080'
```

在客户端访问：

```bash
curl -v http://服务端IP:8080/healthz
```

判断：

```text
服务端 tcpdump 没有任何包：包没到机器，查路由、安全组、上游负载均衡。
看到 SYN 但没有响应：本机防火墙或内核未交给监听进程。
看到三次握手：TCP 通了，继续看 HTTP 和应用日志。
看到 HTTP 请求但应用无日志：可能请求被本机代理、sidecar 或错误进程处理。
```

`tcpdump` 是排障里的证据工具。它不能告诉你业务为什么错，但能告诉你包有没有到。

---

## 六、常见问题

### 1. ss 和 netstat 用哪个？

Linux 上优先用 `ss`，更现代、更快。

### 2. tcpdump 抓不到包怎么办？

检查网卡、过滤条件、网络命名空间、请求是否真的发出。

### 3. curl 500 是否表示网络失败？

不是。500 是 HTTP 响应，说明至少已经收到服务端或网关响应。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 用 dig 查 DNS。
- 用 curl 看 HTTP 细节。
- 用 ss 看监听和连接状态。
- 用 lsof 找端口进程。
- 用 tcpdump 抓包证明流量。
