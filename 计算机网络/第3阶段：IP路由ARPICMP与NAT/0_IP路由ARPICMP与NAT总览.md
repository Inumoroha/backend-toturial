# 0. IP、路由、ARP、ICMP 与 NAT 总览

本阶段目标：理解 IP 地址、子网、路由表、ARP、ICMP、NAT 的作用，并能用命令证明“包往哪里走、下一跳是谁、目标是否可达、为什么容器能访问外网”。

前面你已经知道 MAC、IP、端口分别解决不同问题。本阶段开始进入网络层和局域网转发的核心：

```text
IP 负责主机寻址。
路由表决定下一跳。
ARP 解决同一链路内 IP 到 MAC 的映射。
ICMP 用来传递网络控制信息。
NAT 让私有地址访问公网。
```

---

## 一、本阶段学习顺序

建议按下面顺序学习：

```text
1_IP地址_CIDR_私有地址与公网地址
2_路由表_默认网关与traceroute
3_ARP协议与局域网下一跳
4_ICMP_ping_traceroute与连通性判断
5_NAT原理与Docker容器访问外网
6_综合排障_从域名到目标IP的网络层检查
```

这些内容是后端排障的基础。服务访问失败时，你需要知道：

- DNS 有没有解析出 IP。
- 本机有没有到目标 IP 的路由。
- 目标是否在同一网段。
- 如果不在同一网段，默认网关是谁。
- 端口不通到底是网络不通，还是服务没监听。

---

## 二、本阶段常用命令

```bash
ip addr
ip route
ip route get 8.8.8.8
ip neigh
arp -a
ping 8.8.8.8
traceroute example.com
curl -v http://example.com
docker network inspect bridge
```

Windows 对应：

```powershell
ipconfig
route print
arp -a
ping 8.8.8.8
tracert example.com
Test-NetConnection example.com -Port 443
```

---

## 三、本阶段达标标准

学完本阶段后，你应该能够做到：

- 解释 `192.168.1.10/24` 的含义。
- 区分私有 IP 和公网 IP。
- 看懂 `ip route` 的默认路由。
- 使用 `ip route get` 判断访问目标走哪个网关。
- 使用 `ip neigh` 或 `arp -a` 查看 IP 与 MAC 的映射。
- 解释 `ping` 能证明什么，不能证明什么。
- 解释 NAT 为什么能让多个内网设备共享公网 IP。
- 解释 Docker 容器访问外网为什么通常依赖 NAT。

