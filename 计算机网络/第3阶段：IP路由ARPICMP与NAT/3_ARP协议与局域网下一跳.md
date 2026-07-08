# 3. ARP 协议与局域网下一跳

本节目标：理解 ARP 如何把 IP 地址解析成 MAC 地址，并能通过 `ip neigh`、`arp -a` 和抓包观察 ARP 请求与响应。

---

## 一、ARP 解决什么问题

同一局域网内，主机知道目标 IP 后，还需要知道目标 MAC，才能发送以太网帧。

ARP 的作用就是：

```text
根据 IP 地址找到同一链路内的 MAC 地址。
```

例如：

```text
谁是 192.168.1.1？请告诉 192.168.1.10。
```

网关收到后回复：

```text
192.168.1.1 的 MAC 是 xx:xx:xx:xx:xx:xx。
```

---

## 二、ARP 只负责当前链路

ARP 不会帮你查公网服务器的 MAC。

如果访问 `8.8.8.8`，你的电脑通常只需要知道默认网关的 MAC。

因为当前这一跳是：

```text
本机 -> 默认网关
```

后续每一跳都由对应路由器重新封装链路层帧。

---

## 三、查看 ARP / 邻居缓存

Linux：

```bash
ip neigh
```

或：

```bash
arp -a
```

Windows：

```powershell
arp -a
```

你可能看到：

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

表示本机知道 `192.168.1.1` 的 MAC 地址。

---

## 四、实验：观察网关 ARP

第一步，查看默认网关：

```bash
ip route
```

假设网关是：

```text
192.168.1.1
```

第二步，ping 网关：

```bash
ping 192.168.1.1
```

第三步，查看邻居缓存：

```bash
ip neigh | grep 192.168.1.1
```

你应该能看到网关的 MAC。

---

## 五、用 Wireshark 抓 ARP

Wireshark 显示过滤器：

```text
arp
```

你可能看到：

```text
Who has 192.168.1.1? Tell 192.168.1.10
192.168.1.1 is at aa:bb:cc:dd:ee:ff
```

这就是 ARP 请求和响应。

如果看不到，可以等待缓存过期，或者换一个本地网段内没访问过的目标。

---

## 六、ARP 在后端排障中的位置

后端工程师不常直接处理 ARP，但在这些场景需要知道它：

- 同网段机器偶发访问失败。
- 网关 MAC 异常。
- IP 冲突。
- 虚拟机或容器网络异常。
- ARP 缓存污染或老化。

如果同网段 IP 访问异常，可以查看：

```bash
ip neigh
```

确认邻居状态是否正常。

---

## 补充实验：观察 ARP 从无到有

先查看邻居表：

```bash
ip neigh
```

找一个同网段目标，例如默认网关：

```bash
ip route | grep default
```

删除已有邻居缓存：

```bash
sudo ip neigh flush all
```

再 ping 网关：

```bash
ping -c 1 网关IP
```

重新查看：

```bash
ip neigh
```

你应该看到类似：

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

这表示本机已经知道“网关 IP 对应哪个 MAC 地址”。同网段通信和访问默认网关都离不开这个步骤。

---

## 补充抓包：ARP 请求和响应长什么样

抓 ARP：

```bash
sudo tcpdump -i eth0 -nn arp
```

再访问网关：

```bash
ping -c 1 网关IP
```

典型输出：

```text
ARP, Request who-has 192.168.1.1 tell 192.168.1.10
ARP, Reply 192.168.1.1 is-at aa:bb:cc:dd:ee:ff
```

含义：

```text
本机问：谁是 192.168.1.1？
网关答：我是，我的 MAC 是 aa:bb:cc:dd:ee:ff。
```

如果 ARP 一直只有 request 没有 reply，说明同二层网络里没有设备回应，可能是网段配置错、网关不可达、VLAN 隔离或虚拟网络配置问题。

---

## 七、常见问题

### 1. ARP 是广播吗？

ARP 请求通常是广播，ARP 响应通常是单播。

### 2. ARP 能跨路由器吗？

不能。ARP 只在当前二层广播域内工作。

### 3. 访问公网为什么 ARP 表里没有公网服务器？

因为本机只需要把包交给默认网关，不需要知道公网服务器的 MAC。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 ARP 解决的问题。
- 使用 `ip neigh` 或 `arp -a` 查看邻居缓存。
- 说明为什么跨网段访问时 ARP 查的是网关 MAC。
- 使用 Wireshark 过滤 `arp` 观察请求和响应。
