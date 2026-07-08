# 4. ICMP、ping、traceroute 与连通性判断

本节目标：理解 ICMP 的作用，知道 `ping` 和 `traceroute` 能证明什么、不能证明什么，避免把“ping 通”误认为“服务一定正常”。

---

## 一、ICMP 是什么

ICMP 是网络层控制协议，用来传递网络状态和错误信息。

常见用途：

- `ping` 使用 ICMP Echo Request / Echo Reply。
- 路由不可达时可能返回 ICMP Destination Unreachable。
- 路径 MTU 发现可能依赖 ICMP Fragmentation Needed。

ICMP 不是业务协议，但对排障非常重要。

---

## 二、ping 能证明什么

执行：

```bash
ping example.com
```

如果成功，说明：

- 域名大概率解析成功。
- 目标 IP 或中间设备回应了 ICMP。
- 网络层往返大致可达。
- 可以看到延迟和丢包线索。

---

## 三、ping 不能证明什么

`ping` 成功不代表：

- TCP 端口开放。
- HTTP 服务正常。
- TLS 证书正确。
- 业务接口可用。
- 防火墙没有限制 TCP。

所以服务排障不能只停在 ping。

例如：

```bash
ping 服务器IP
curl -v http://服务器IP:8080
```

两者验证的是不同层面。

---

## 四、为什么 ping 失败但网站能访问

有些服务器或云厂商会禁用 ICMP。

这时：

```bash
ping example.com
```

可能失败，但：

```bash
curl -v https://example.com
```

仍然成功。

所以 ping 失败只能说明 ICMP 没有成功，不一定说明 TCP/HTTPS 不可用。

---

## 五、测试 TCP 端口

使用 nc：

```bash
nc -vz example.com 443
```

或 curl：

```bash
curl -v https://example.com
```

Windows：

```powershell
Test-NetConnection example.com -Port 443
```

这些比 ping 更适合判断服务端口是否可连接。

---

## 六、traceroute 的作用

```bash
traceroute example.com
```

Windows：

```powershell
tracert example.com
```

它可以显示到目标的大致路径和每一跳延迟。

适合排查：

- 跨地域延迟高。
- 某一跳之后不通。
- 路径绕远。
- 运营商链路异常。

但要注意，traceroute 结果不是绝对完整证据，因为中间设备可能不响应探测。

---

## 七、后端排障顺序建议

当接口访问失败时，可以按顺序检查：

```bash
dig 域名
ping 目标IP
ip route get 目标IP
nc -vz 目标IP 端口
curl -v URL
tcpdump 抓包
```

这样可以逐层缩小问题范围。

---

## 补充实验：ping 通但端口不通

选择一台能 ping 通的机器后，继续测试端口：

```bash
ping -c 3 目标IP
nc -vz 目标IP 80
nc -vz 目标IP 443
```

你可能得到：

```text
ping 成功。
80 refused。
443 succeeded。
```

这说明：

```text
主机可达。
80 端口没有服务或被拒绝。
443 端口可建立 TCP 连接。
```

所以 `ping` 只能证明 ICMP 层面有响应，不能证明 HTTP、数据库、Redis、gRPC 都可用。

---

## 补充判断：ICMP 被禁不等于业务不通

很多生产环境会禁止 ICMP：

```bash
ping 目标IP
```

可能失败，但：

```bash
curl -I https://目标域名
```

仍然成功。

排障结论要写清楚：

```text
ICMP 无响应，但 TCP 443 建连成功，HTTPS 返回 200。
因此不能判定目标主机不可达，只能说明 ICMP 不可用或被过滤。
```

后端排障时，业务协议的测试优先级通常高于 ping。

---

## 八、常见问题

### 1. ping 通但 curl Connection refused？

目标主机可达，但目标 TCP 端口没有服务监听或被拒绝。

### 2. ping 通但 curl timeout？

可能是端口被防火墙丢弃，或者服务没有响应。

### 3. traceroute 中间出现星号怎么办？

不一定是故障，中间路由器可能不回复探测报文。

---

## 九、本节达标标准

学完本节后，你应该能够做到：

- 解释 ICMP 的基本作用。
- 说明 ping 能证明什么、不能证明什么。
- 使用 `nc` 或 `Test-NetConnection` 测试 TCP 端口。
- 使用 `traceroute` 或 `tracert` 查看路径。
- 在排障中区分网络层可达和应用服务可用。
