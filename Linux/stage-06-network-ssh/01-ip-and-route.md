# 01：IP、网卡与路由

本节目标：学会查看本机 IP、网卡状态、默认路由，并理解“本机能不能出网”从哪里开始排查。

本节命令顺序：

```text
1. ip addr
2. ip addr show
3. hostname -I
4. ip route
5. ip route get
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-06
```

---

## 1. IP 地址是什么

IP 地址用于在网络中标识一台主机或一个网络接口。

常见类型：

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| IPv4 | `192.168.1.10` | 最常见 |
| IPv6 | `fe80::...` | 新一代地址 |
| 回环地址 | `127.0.0.1` | 指向本机 |

常见私有 IPv4 地址段：

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

这些通常用于家庭、公司、虚拟机、容器内部网络。

---

## 2. ip addr：查看网卡和 IP

执行：

```bash
ip addr
```

你会看到类似：

```text
1: lo: <LOOPBACK,UP,LOWER_UP>
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

重点看：

| 字段 | 含义 |
| --- | --- |
| `lo` | 回环网卡，本机内部使用 |
| `eth0` | 常见网卡名称 |
| `inet` | IPv4 地址 |
| `inet6` | IPv6 地址 |
| `UP` | 网卡已启用 |

只查看某个网卡：

```bash
ip addr show eth0
```

不同系统网卡名可能不同，例如 `ens33`、`enp0s3`。

---

## 3. hostname -I：快速查看本机 IP

执行：

```bash
hostname -I
```

它会输出本机 IP 地址，适合快速查看。

注意：

```text
如果一台机器有多个网卡，可能输出多个 IP。
```

---

## 4. ip route：查看路由

执行：

```bash
ip route
```

你可能看到：

```text
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

重点看 `default`：

```text
default via 192.168.1.1 dev eth0
```

含义：

```text
访问不知道怎么走的网络时，交给 192.168.1.1 这个网关。
```

如果没有默认路由，通常无法访问公网。

---

## 5. ip route get：查看访问目标走哪条路

查看访问公网 IP 会走哪条路：

```bash
ip route get 8.8.8.8
```

查看访问某个域名解析后的 IP：

```bash
ip route get 1.1.1.1
```

输出里重点看：

| 字段 | 含义 |
| --- | --- |
| `via` | 下一跳网关 |
| `dev` | 使用哪个网卡 |
| `src` | 本机使用哪个源 IP |

---

## 6. 本节练习

执行并记录输出：

```bash
ip addr
hostname -I
ip route
ip route get 8.8.8.8
```

回答：

1. 你的主要网卡名是什么？
2. 你的本机 IPv4 地址是什么？
3. 默认网关是什么？
4. 访问 `8.8.8.8` 使用哪个网卡？

---

## 7. 本节小结

必须记住：

```bash
ip addr
ip addr show eth0
hostname -I
ip route
ip route get 8.8.8.8
```

核心思路：

```text
先确认本机有没有 IP，再确认有没有默认路由。
```

完成后进入下一节：`02-connectivity-and-dns.md`。

