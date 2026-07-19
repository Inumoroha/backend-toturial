# 5. MTU、MSS 与网络分片问题

本节目标：理解 MTU 和 MSS 的含义，知道为什么大请求、大响应、VPN、Docker、Kubernetes 网络中可能出现“小包正常，大包失败”的问题。

MTU/MSS 初学时容易被忽略，因为平时写 Go 业务代码几乎不会直接设置它们。但在线上排障中，它们偶尔会造成非常隐蔽的问题。

---

## 一、MTU 是什么

MTU 是 Maximum Transmission Unit，表示一个链路层帧能承载的最大数据大小。

以太网常见 MTU 是：

```text
1500 字节
```

查看本机 MTU：

```bash
ip link
```

你可能看到：

```text
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
```

这里的 `mtu 1500` 就是该网卡的 MTU。

---

## 二、MSS 是什么

MSS 是 Maximum Segment Size，表示 TCP 一个段中最多能携带多少应用层数据。

常见计算：

```text
MSS = MTU - IP 头 - TCP 头
```

IPv4 常见 IP 头 20 字节，TCP 头 20 字节。

所以：

```text
1500 - 20 - 20 = 1460
```

这就是为什么你经常看到 TCP MSS 为 1460。

---

## 三、为什么不能无限发大包

网络链路对单个帧大小有限制。如果 IP 包超过路径中某一段链路的 MTU，可能发生：

- IP 分片。
- 发送方根据路径 MTU 调整包大小。
- 中间设备丢弃过大的包。

如果分片或路径 MTU 发现出现问题，就可能发生：

```text
小请求正常。
大响应失败。
上传文件卡住。
VPN 内访问某些接口超时。
容器网络中部分请求异常。
```

---

## 四、Docker 和 VPN 为什么容易遇到 MTU 问题

Docker overlay、VPN、隧道网络通常会在原始包外面再包一层头部。

例如：

```text
原始 IP 包
外层隧道头
外层 IP 头
```

额外头部会占用空间，导致实际可用 MTU 变小。

如果应用仍然发送接近 1500 的包，中间链路可能装不下。

---

## 五、实验：查看 MTU

查看所有网卡：

```bash
ip link
```

只看某个网卡：

```bash
ip link show eth0
```

如果你使用 Docker，也可以看 Docker bridge：

```bash
ip link show docker0
```

记录每个网卡的 MTU。

---

## 六、实验：测试路径 MTU

Linux：

```bash
ping -M do -s 1472 8.8.8.8
```

参数说明：

- `-M do`：不允许分片。
- `-s 1472`：ICMP payload 大小。

为什么是 1472？

```text
1472 + 20 字节 IP 头 + 8 字节 ICMP 头 = 1500
```

如果失败，可以逐步减小：

```bash
ping -M do -s 1400 8.8.8.8
ping -M do -s 1300 8.8.8.8
```

Windows：

```powershell
ping 8.8.8.8 -f -l 1472
```

其中：

- `-f`：不允许分片。
- `-l`：指定 payload 大小。

---

## 七、Go 后端中如何遇到 MTU 问题

典型现象：

```text
GET /health 正常。
GET /large-report 超时。
小 JSON 正常。
大 JSON 或文件上传失败。
内网正常，VPN 访问失败。
裸机正常，容器网络异常。
```

排查时可以对比：

- 小响应和大响应。
- HTTP 和 HTTPS。
- 宿主机访问和容器访问。
- 公司网络和手机热点。
- VPN 开启和关闭。

MTU 问题不常见，但一旦遇到，单看应用日志很难定位。

---

## 八、如何缓解 MTU 相关问题

常见方向：

- 调整网卡或隧道 MTU。
- 调整 Docker / Kubernetes CNI 的 MTU 配置。
- 避免单个响应过大。
- 使用分页或流式传输。
- 确保 ICMP Fragmentation Needed 没被防火墙拦截。

应用层不一定直接解决 MTU，但可以减少大包压力，例如：

```text
分页查询。
压缩响应。
分块上传。
流式下载。
```

---

## 九、常见问题

### 1. MTU 和 MSS 是一回事吗？

不是。MTU 是链路层最大传输单元，MSS 是 TCP 数据段最大应用数据大小。

### 2. 为什么常见 MSS 是 1460？

因为以太网 MTU 1500，减去 IPv4 头 20 字节和 TCP 头 20 字节，剩下 1460。

### 3. 为什么小请求正常不能证明网络完全正常？

小请求可能不会触发分片或大包传输问题，大响应才会暴露 MTU、拥塞、缓冲区等问题。

### 4. Go 代码里要不要手动设置 MSS？

通常不需要。后端工程师更需要知道如何识别 MTU 问题，以及如何从应用层减少大响应压力。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 使用 `ip link` 查看 MTU。
- 解释 MTU 和 MSS 的区别。
- 解释为什么常见 MSS 是 1460。
- 使用 `ping -M do -s` 或 Windows `ping -f -l` 测试路径 MTU。
- 在“小请求正常，大请求失败”时把 MTU 纳入排查范围。
- 解释 Docker、VPN、隧道网络为什么更容易遇到 MTU 问题。

完成本节后，第 2 阶段就达标了。下一阶段可以继续学习 IP、路由、ARP、ICMP 和 NAT。

