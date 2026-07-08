# 5. 防火墙、安全组、iptables 与 nftables

本节目标：理解防火墙和云安全组如何影响服务访问，掌握基础查看命令。

---

## 一、访问链路可能经过多层过滤

```text
客户端
公司网络
云安全组
服务器防火墙
Docker 转发规则
Go 服务
```

端口不通时，不能只看 Go 服务是否启动。

---

## 二、查看 iptables

```bash
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
```

说明：

- `filter` 表控制允许或拒绝。
- `nat` 表常见地址转换和端口转发。

---

## 三、查看 nftables

```bash
sudo nft list ruleset
```

新系统可能使用 nftables。

---

## 四、云安全组

云服务器公网访问失败时，要检查：

- 安全组入站规则。
- 目标端口是否开放。
- 来源 IP 是否限制。
- 协议是 TCP 还是 UDP。

本机 curl 成功、公网失败，常见就是安全组没放行。

---

## 五、排查模板

```bash
ss -lntp | grep 8080
curl http://127.0.0.1:8080
curl http://服务器内网IP:8080
sudo iptables -L -n -v
```

然后检查云控制台安全组。

---

## 补充判断：refused 和 timeout 的不同方向

访问端口失败时，错误文案很重要。

如果是：

```text
Connection refused
```

常见含义：

```text
目标 IP 可达。
目标机器明确返回拒绝。
端口没有监听，或有规则主动 REJECT。
```

如果是：

```text
Connection timed out
```

常见含义：

```text
SYN 发出后没有回应。
安全组、防火墙、路由 ACL 可能丢弃。
目标机器可能不可达。
```

用抓包验证：

```bash
sudo tcpdump -i any -nn host 目标IP and port 目标端口
```

判断：

```text
看到 SYN 后收到 RST：更接近 refused。
只看到 SYN 重传：更接近 timeout。
```

---

## 补充流程：本机成功但公网失败

如果：

```bash
curl http://127.0.0.1:8080
```

成功，但公网访问失败，按顺序检查：

```bash
ss -lntp | grep ':8080'
curl http://服务器内网IP:8080
sudo iptables -L -n -v
sudo nft list ruleset
```

然后检查云安全组：

```text
入站规则是否允许目标端口。
来源 IP 是否包含你的客户端出口 IP。
协议是否是 TCP。
绑定的安全组是否是当前实例使用的安全组。
```

很多事故不是服务没启动，而是安全组改到了另一台机器、另一个端口或另一个来源网段。

---

## 补充安全提醒：不要为了排障直接全放开

排障时不要随手把安全组改成：

```text
0.0.0.0/0 -> all ports
```

更稳妥的方式是：

```text
只放行目标端口。
只放行测试客户端出口 IP。
设置临时规则并记录过期时间。
验证后及时收回。
```

如果必须临时放开，也要记录：

```text
谁改的。
为什么改。
改了什么。
什么时候恢复。
验证结果。
```

网络排障不能以制造安全风险为代价。

---

## 六、常见问题

### 1. Connection refused 是防火墙吗？

不一定。通常是目标端口没有监听或主动拒绝。防火墙丢弃更常表现为 timeout。

### 2. 本机访问成功，公网失败优先看什么？

监听地址、防火墙、安全组。

### 3. Docker 会修改 iptables 吗？

会。Docker 端口映射通常依赖 iptables/nftables 规则。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 查看 iptables/nftables 规则。
- 解释安全组和系统防火墙的区别。
- 排查本机成功但公网失败的问题。
- 区分 refused 和 timeout 的常见含义。
