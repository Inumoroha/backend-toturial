# 8. 浏览器访问 HTTPS 网站完整复盘

本节目标：把 DNS、TCP、TLS、HTTP 串起来，完整解释浏览器访问 `https://example.com` 的过程。

---

## 一、完整流程

浏览器访问：

```text
https://example.com/articles/1
```

大致过程：

```text
1. 解析 URL。
2. 查询 DNS，得到 IP。
3. 根据路由表找到下一跳。
4. ARP 获取下一跳 MAC。
5. 建立 TCP 连接。
6. TLS 握手，验证证书，协商密钥。
7. 发送 HTTP 请求。
8. 服务端返回 HTTP 响应。
9. 浏览器处理状态码、Header、Cookie、缓存和 body。
10. 连接复用或关闭。
```

---

## 二、用命令拆解

DNS：

```bash
dig example.com
```

路由：

```bash
ip route get 目标IP
```

TCP/TLS/HTTP：

```bash
curl -v https://example.com/articles/1
```

抓包：

```bash
sudo tcpdump -i any host 目标IP -nn
```

---

## 三、后端视角

后端服务需要关注：

- DNS 是否稳定。
- TCP 连接是否可建立。
- TLS 证书是否有效。
- 网关是否正确转发 Header。
- 服务是否设置超时。
- HTTP 状态码是否表达正确。
- 日志是否能串起 request id。

---

## 四、常见失败点

| 阶段 | 失败表现 |
| --- | --- |
| DNS | 域名解析失败 |
| 路由 | no route to host |
| TCP | connection refused / timeout |
| TLS | certificate error |
| HTTP | 4xx / 5xx |
| 业务 | 状态码 200 但业务错误 |

---

## 补充演练：用一张表记录每一层证据

排查一次 HTTPS 访问问题时，可以按下面表格记录：

| 阶段 | 命令 | 正常证据 | 异常方向 |
| --- | --- | --- | --- |
| DNS | `dig example.com` | 有 A/AAAA 记录 | DNS、hosts、递归解析 |
| 路由 | `ip route get IP` | 有 dev、src、via | 默认路由、VPN、多网卡 |
| TCP | `curl -v` | Connected | refused、timeout、安全组 |
| TLS | `openssl s_client` | Verify ok | 证书、CA、SNI |
| HTTP | `curl -i` | 合理状态码 | 4xx、5xx、网关 |
| 业务 | 服务日志 | request id 可追踪 | 参数、数据库、下游 |

示例结论不要写成：

```text
接口不通。
```

而要写成：

```text
DNS 解析到 10.20.1.15 正常。
请求方路由从 eth0 出口正常。
tcpdump 只看到 SYN 重传，没有 SYN-ACK。
服务端无访问日志。
判断问题在 TCP 建连前后，优先检查安全组或目标监听。
```

这样别人才能复核你的判断。

---

## 补充后端视角：一次请求经过哪些组件

真实生产链路可能是：

```text
浏览器
-> CDN
-> WAF
-> 负载均衡
-> Nginx / API 网关
-> Go 服务
-> Redis / PostgreSQL / 下游服务
```

每一层都可能设置：

```text
超时。
最大 Header。
最大 Body。
TLS 策略。
限流。
访问日志。
错误码转换。
```

所以浏览器看到一个 `504`，并不一定是 Go 服务自己返回的。你需要看是哪一层生成了这个响应：

```text
响应 Header 里的 Server。
网关 access log。
Go 服务 access log。
链路追踪 request id。
```

能把这些层次串起来，才算真正理解“浏览器访问 HTTPS 网站完整流程”。

---

## 补充练习：把一次访问画成时序

请尝试把下面访问：

```text
https://api.example.com/users/1
```

写成自己的时序图：

```text
浏览器 -> DNS：查询 api.example.com
DNS -> 浏览器：返回 1.2.3.4
浏览器 -> 1.2.3.4:443：TCP 三次握手
浏览器 -> 1.2.3.4:443：TLS ClientHello，携带 SNI
服务端 -> 浏览器：证书链
浏览器：验证证书
浏览器 -> 服务端：HTTP GET /users/1
服务端 -> 浏览器：HTTP 200 + JSON
```

然后在每一行后面标出可用命令：

```text
DNS：dig
TCP/TLS/HTTP：curl -v
证书：openssl s_client
路由：ip route get
抓包：tcpdump
```

这份时序图能帮助你把零散知识变成完整链路。

---

## 五、常见问题

### 1. HTTPS 访问失败一定是证书问题吗？

不一定。DNS、TCP、TLS、HTTP 任一阶段都可能失败，要按阶段排查。

### 2. 浏览器能访问，Go 程序访问失败是什么原因？

可能是 Go 运行环境 DNS、代理、根证书、TLS 配置、超时配置和浏览器不同。

### 3. HTTP 状态码 200 是否代表业务成功？

不一定。很多接口会在 body 中返回业务错误码，所以还要看业务协议。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 口述 HTTPS 访问完整流程。
- 用 `dig`、`ip route get`、`curl -v` 分段验证。
- 区分 DNS、TCP、TLS、HTTP、业务错误。
- 从后端角度说明每一阶段可能如何影响接口可用性。
