# 9. 网络命名空间、veth、bridge 与 NAT 手把手实验

本节目标：不用 Docker，手动创建一个“迷你容器网络”，理解 network namespace、veth pair、Linux bridge、默认网关和 NAT 的关系。

Docker 网络看起来复杂，本质上离不开这些 Linux 基础能力：

```text
network namespace 隔离网络协议栈。
veth pair 像一根虚拟网线。
bridge 像一个虚拟交换机。
iptables/nftables 做 NAT 和转发。
```

本节实验建议在 Linux 或 WSL2 Ubuntu 中执行。需要 root 权限。

---

## 一、实验拓扑

最终拓扑：

```text
ns1(10.200.1.2/24)
  |
veth-ns1
  |
veth-host
  |
br-demo(10.200.1.1/24)
  |
宿主机默认网卡
  |
外网
```

目标：

- `ns1` 能 ping 通网关 `10.200.1.1`。
- `ns1` 能通过宿主机 NAT 访问外网。
- 你能用命令解释每一跳发生了什么。

---

## 二、清理旧环境

如果之前做过实验，先清理：

```bash
sudo ip netns del ns1 2>/dev/null || true
sudo ip link del br-demo 2>/dev/null || true
```

这两条命令的作用：

- 删除旧 namespace。
- 删除旧 bridge。

`2>/dev/null || true` 是为了让“不存在”不影响继续执行。

---

## 三、创建 network namespace

```bash
sudo ip netns add ns1
```

查看：

```bash
ip netns list
```

预期：

```text
ns1
```

进入 namespace 执行命令：

```bash
sudo ip netns exec ns1 ip addr
```

你会看到里面有独立的 `lo`。这说明 `ns1` 有自己的网络设备视图。

启用 loopback：

```bash
sudo ip netns exec ns1 ip link set lo up
```

---

## 四、创建 veth pair

```bash
sudo ip link add veth-host type veth peer name veth-ns1
```

查看：

```bash
ip link show | grep veth
```

veth pair 可以理解为一根虚拟网线：

```text
从 veth-host 发出的包，会从 veth-ns1 出来。
从 veth-ns1 发出的包，会从 veth-host 出来。
```

把 `veth-ns1` 放进 `ns1`：

```bash
sudo ip link set veth-ns1 netns ns1
```

此时宿主机只看得到 `veth-host`，`ns1` 里能看到 `veth-ns1`。

验证：

```bash
ip link show veth-host
sudo ip netns exec ns1 ip link show veth-ns1
```

---

## 五、创建 bridge

```bash
sudo ip link add br-demo type bridge
sudo ip addr add 10.200.1.1/24 dev br-demo
sudo ip link set br-demo up
```

把宿主机侧 veth 接到 bridge：

```bash
sudo ip link set veth-host master br-demo
sudo ip link set veth-host up
```

解释：

- `br-demo` 像一个虚拟交换机。
- `veth-host` 像插到交换机上的网线。
- `10.200.1.1` 是这个小网络的网关地址。

---

## 六、配置 namespace 内 IP

```bash
sudo ip netns exec ns1 ip addr add 10.200.1.2/24 dev veth-ns1
sudo ip netns exec ns1 ip link set veth-ns1 up
```

查看：

```bash
sudo ip netns exec ns1 ip addr show veth-ns1
```

预期看到：

```text
inet 10.200.1.2/24
```

---

## 七、测试 namespace 到网关

```bash
sudo ip netns exec ns1 ping -c 3 10.200.1.1
```

如果成功，说明：

```text
ns1 -> veth-ns1 -> veth-host -> br-demo
```

这条二层链路已经打通。

查看 ARP：

```bash
sudo ip netns exec ns1 ip neigh
```

你应该能看到 `10.200.1.1` 对应的 MAC 地址。

---

## 八、配置默认路由

现在 `ns1` 还不知道访问外网该走哪里。

添加默认路由：

```bash
sudo ip netns exec ns1 ip route add default via 10.200.1.1
```

查看：

```bash
sudo ip netns exec ns1 ip route
```

预期：

```text
default via 10.200.1.1 dev veth-ns1
10.200.1.0/24 dev veth-ns1 proto kernel scope link src 10.200.1.2
```

这一步对应 Docker 容器里的默认网关。

---

## 九、打开宿主机 IP 转发

```bash
cat /proc/sys/net/ipv4/ip_forward
```

如果是 `0`，执行：

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

含义：

```text
允许 Linux 主机转发不同网卡之间的 IPv4 包。
```

如果不打开，宿主机不会像路由器一样帮 `ns1` 转发外网流量。

---

## 十、配置 NAT

先找到宿主机出外网的网卡：

```bash
ip route get 8.8.8.8
```

输出里看 `dev`，假设是 `eth0`。

添加 NAT：

```bash
sudo iptables -t nat -A POSTROUTING -s 10.200.1.0/24 -o eth0 -j MASQUERADE
```

如果你的外网网卡不是 `eth0`，要替换成真实网卡名。

这条规则的意思：

```text
来自 10.200.1.0/24 的包，如果从 eth0 出去，就把源地址伪装成宿主机 eth0 的地址。
```

这就是容器访问外网最核心的 NAT 逻辑。

---

## 十一、测试访问外网

```bash
sudo ip netns exec ns1 ping -c 3 8.8.8.8
```

如果成功，说明 IP 层和 NAT 已经通。

再测试 DNS：

```bash
sudo ip netns exec ns1 ping -c 3 example.com
```

如果 IP 能通、域名不通，说明 DNS 没配。

给 namespace 配 DNS：

```bash
sudo mkdir -p /etc/netns/ns1
echo "nameserver 8.8.8.8" | sudo tee /etc/netns/ns1/resolv.conf
```

再试：

```bash
sudo ip netns exec ns1 ping -c 3 example.com
```

---

## 十二、抓包观察

在 bridge 上抓：

```bash
sudo tcpdump -i br-demo -nn icmp
```

另一个终端执行：

```bash
sudo ip netns exec ns1 ping -c 3 8.8.8.8
```

你会看到源地址是：

```text
10.200.1.2
```

在外网网卡抓：

```bash
sudo tcpdump -i eth0 -nn host 8.8.8.8 and icmp
```

你会看到源地址变成宿主机外网地址。

这就是 NAT 前后源地址变化。

---

## 十三、对照 Docker 网络

现在回头看 Docker bridge：

```bash
docker network inspect bridge
ip addr show docker0
iptables -t nat -S | grep MASQUERADE
```

你会发现 Docker 做的事情和本节非常像：

```text
为容器创建 network namespace。
创建 veth pair。
把宿主机侧 veth 接到 docker0 bridge。
给容器配置 IP 和默认网关。
配置 NAT，让容器访问外网。
```

理解这一套后，Docker 网络就不再是黑盒。

---

## 十四、清理环境

实验结束后清理：

```bash
sudo ip netns del ns1
sudo ip link del br-demo
sudo iptables -t nat -D POSTROUTING -s 10.200.1.0/24 -o eth0 -j MASQUERADE
sudo rm -rf /etc/netns/ns1
```

注意最后一条 iptables 删除命令里的网卡名要和你添加时一致。

---

## 十五、常见问题

### 1. ping 网关不通怎么办？

按顺序检查：

```bash
sudo ip netns exec ns1 ip addr
sudo ip netns exec ns1 ip link
ip addr show br-demo
bridge link
```

常见原因：

```text
veth-ns1 没 up。
veth-host 没接到 br-demo。
br-demo 没 up。
IP 地址写错网段。
```

### 2. ping 8.8.8.8 不通怎么办？

检查：

```bash
sudo ip netns exec ns1 ip route
cat /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -S
ip route get 8.8.8.8
```

常见原因：

```text
namespace 没默认路由。
宿主机没开启 ip_forward。
NAT 规则里的出口网卡写错。
宿主机本身不能访问外网。
```

### 3. IP 能通但域名不通怎么办？

这是 DNS 问题。

检查：

```bash
sudo ip netns exec ns1 cat /etc/resolv.conf
```

给 namespace 配置 `/etc/netns/ns1/resolv.conf` 后再试。

### 4. WSL2 可以做这个实验吗？

多数 WSL2 Ubuntu 可以完成 namespace、veth、bridge 实验，但和 Windows 主机网络的边界可能有差异。如果 NAT 或外网访问失败，优先在原生 Linux 虚拟机中复现。

---

## 十六、本节达标标准

学完本节后，你应该能够做到：

- 创建和删除 Linux network namespace。
- 创建 veth pair，并把一端移动到 namespace。
- 创建 Linux bridge，并把 veth 接入 bridge。
- 给 namespace 配置 IP、默认路由和 DNS。
- 开启宿主机 IPv4 转发。
- 使用 iptables MASQUERADE 实现 NAT。
- 用 tcpdump 观察 NAT 前后的源地址变化。
- 解释 Docker bridge 网络和本实验的对应关系。
- 根据命令输出定位 namespace 网络不通的原因。

