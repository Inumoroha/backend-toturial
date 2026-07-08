# 02：连通性与 DNS 排查

本节目标：学会用 `ping` 判断网络连通性，用 `traceroute` 查看路径，用 `dig`/`nslookup` 排查域名解析。

本节命令顺序：

```text
1. ping
2. ping -c
3. traceroute
4. dig
5. dig +short
6. nslookup
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-06
```

---

## 1. ping：测试基础连通性

测试公网 IP：

```bash
ping 8.8.8.8
```

停止：

```text
Ctrl + C
```

只发送 4 次：

```bash
ping -c 4 8.8.8.8
```

测试域名：

```bash
ping -c 4 example.com
```

注意：

```text
有些服务器会禁用 ICMP，所以 ping 不通不一定代表服务不可用。
```

---

## 2. 通过 ping 区分网络和 DNS 问题

先 ping 公网 IP：

```bash
ping -c 4 8.8.8.8
```

再 ping 域名：

```bash
ping -c 4 example.com
```

判断：

| 现象 | 可能原因 |
| --- | --- |
| IP 和域名都通 | 基础网络正常 |
| IP 通，域名不通 | DNS 可能有问题 |
| IP 不通，域名也不通 | 网络、路由、网关可能有问题 |

---

## 3. traceroute：查看经过的网络路径

如果没有安装：

```bash
sudo apt install traceroute
```

执行：

```bash
traceroute example.com
```

它会显示从本机到目标主机经过哪些路由节点。

如果看到很多 `* * *`，可能是中间设备不响应探测包，不一定代表网络完全断开。

---

## 4. dig：查询 DNS

如果没有安装：

```bash
sudo apt install dnsutils
```

查询域名：

```bash
dig example.com
```

只看简短结果：

```bash
dig +short example.com
```

查询指定记录类型：

```bash
dig A example.com
dig AAAA example.com
dig MX example.com
```

常见记录：

| 类型 | 含义 |
| --- | --- |
| `A` | IPv4 地址 |
| `AAAA` | IPv6 地址 |
| `CNAME` | 别名 |
| `MX` | 邮件服务器 |
| `NS` | 域名服务器 |

---

## 5. 指定 DNS 服务器查询

使用 Google DNS：

```bash
dig @8.8.8.8 example.com
```

使用 Cloudflare DNS：

```bash
dig @1.1.1.1 example.com
```

如果系统默认 DNS 查不到，但指定 `8.8.8.8` 能查到，说明本机或网络的 DNS 配置可能有问题。

---

## 6. nslookup：另一种 DNS 查询工具

查询域名：

```bash
nslookup example.com
```

指定 DNS：

```bash
nslookup example.com 8.8.8.8
```

`dig` 输出更详细，`nslookup` 更简单直接。二者会一个就够用，建议优先熟悉 `dig`。

---

## 7. 本节练习

依次执行：

```bash
ping -c 4 8.8.8.8
ping -c 4 example.com
traceroute example.com
dig +short example.com
dig @8.8.8.8 example.com
nslookup example.com
```

记录：

1. 公网 IP 是否能 ping 通？
2. 域名是否能解析？
3. `dig +short` 返回了什么？
4. `traceroute` 大概经过了几跳？

---

## 8. 本节小结

必须记住：

```bash
ping -c 4 8.8.8.8
ping -c 4 example.com
traceroute example.com
dig example.com
dig +short example.com
dig @8.8.8.8 example.com
nslookup example.com
```

核心思路：

```text
IP 能通但域名不通，优先查 DNS。
```

完成后进入下一节：`03-http-curl-wget.md`。

