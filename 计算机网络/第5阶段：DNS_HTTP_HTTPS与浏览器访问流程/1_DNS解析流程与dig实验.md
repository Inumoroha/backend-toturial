# 1. DNS 解析流程与 dig 实验

本节目标：理解域名如何解析成 IP，掌握 `dig` 的基本用法，并知道 DNS 问题如何影响 Go 后端请求。

---

## 一、DNS 解决什么问题

人更容易记住：

```text
api.example.com
```

机器通信需要：

```text
目标 IP 地址
```

DNS 的作用就是把域名解析成 IP 或其他记录。

---

## 二、常见 DNS 记录

| 记录 | 作用 |
| --- | --- |
| A | 域名到 IPv4 |
| AAAA | 域名到 IPv6 |
| CNAME | 域名别名 |
| MX | 邮件服务器 |
| TXT | 文本验证 |
| NS | 权威 DNS 服务器 |

后端最常见的是 A、AAAA、CNAME。

---

## 三、解析流程

大致流程：

```text
应用查询本机缓存。
查询 hosts。
请求递归 DNS。
递归 DNS 查询根、顶级域、权威 DNS。
返回结果并缓存。
```

通常应用不直接查询根服务器，而是交给配置好的递归 DNS。

---

## 四、dig 实验

查询：

```bash
dig example.com
```

只看结果：

```bash
dig +short example.com
```

查询 CNAME：

```bash
dig www.example.com CNAME
```

追踪解析：

```bash
dig +trace example.com
```

指定 DNS：

```bash
dig @8.8.8.8 example.com
```

---

## 五、TTL 与缓存

TTL 表示记录可以缓存多久。

TTL 短：

- 切换快。
- DNS 查询多。

TTL 长：

- 查询少。
- 变更慢。

域名迁移、灰度切流时，TTL 很重要。

---

## 六、Go 后端中的 DNS

Go 请求某个域名前，通常先解析 DNS。

如果 DNS 慢：

```text
服务端可能完全没有请求日志。
tcpdump 可能看不到目标 IP 的 SYN。
应用层看到的是请求总耗时变长。
```

排查：

```bash
cat /etc/resolv.conf
dig api.example.com
dig @8.8.8.8 api.example.com
```

---

## 补充实验：对比 hosts、本机 DNS 与指定 DNS

DNS 排障时不要只执行一次 `dig` 就下结论。建议按下面顺序对比。

第一步，看系统 hosts：

```bash
cat /etc/hosts
```

Windows：

```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts
```

如果 hosts 中写了：

```text
127.0.0.1 api.example.com
```

应用可能会优先使用这个结果，而不是查询公网 DNS。

第二步，看系统配置的 DNS：

```bash
cat /etc/resolv.conf
```

你会看到类似：

```text
nameserver 172.20.0.1
```

第三步，用系统默认 DNS 查询：

```bash
dig api.example.com
```

第四步，指定公共 DNS 查询：

```bash
dig @8.8.8.8 api.example.com
dig @1.1.1.1 api.example.com
```

如果默认 DNS 和公共 DNS 结果不同，说明问题可能在：

```text
公司内网 DNS。
云厂商私有 DNS。
容器 DNS。
本地 DNS 缓存。
域名正在灰度切换。
```

排障时要把结果记下来：

```text
查询位置：本机 / 容器 / 服务器
使用 DNS：默认 / 8.8.8.8 / 公司 DNS
解析结果：
TTL：
```

---

## 补充实验：容器里的 DNS 和宿主机可能不同

启动一个临时容器：

```bash
docker run --rm -it alpine sh
```

在容器里执行：

```sh
cat /etc/resolv.conf
nslookup example.com
```

再在宿主机执行：

```bash
cat /etc/resolv.conf
dig example.com
```

对比两边的 DNS 服务器和解析结果。

如果 Go 服务跑在容器里，那么你应该优先在容器内排查 DNS，而不是只在宿主机排查。线上常见情况是：

```text
宿主机解析正常。
容器内解析失败。
Kubernetes Pod 内解析失败。
CoreDNS 异常导致服务发现失败。
```

这时服务端不会有访问日志，因为请求还没有走到 TCP 建连阶段。

---

## 七、常见问题

### 1. DNS 失败为什么服务端没日志？

因为请求还没连接到服务端，失败发生在连接之前。

### 2. CNAME 是 IP 吗？

不是。CNAME 是别名，最终还要解析到 A 或 AAAA。

### 3. 修改 DNS 后为什么有人还访问旧 IP？

可能是 TTL 缓存未过期。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 使用 `dig` 查询 A、CNAME 记录。
- 解释递归 DNS 和权威 DNS。
- 解释 TTL 对变更生效的影响。
- 把 DNS 纳入请求慢和请求失败排查。
