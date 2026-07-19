# 5. 使用 Wireshark 与 tcpdump 做第一次抓包

本节目标：完成第一次抓包实验，观察 ping、DNS 和 HTTP 请求。你不需要一开始就看懂每个字段，但要建立一个关键直觉：网络请求不是抽象的，它最终会变成真实报文。

后端排障中，抓包常用于回答这些问题：

```text
请求有没有发出去？
对方有没有回包？
TCP 握手有没有完成？
DNS 查询有没有响应？
连接是被谁重置的？
```

---

## 一、Wireshark 和 tcpdump 的区别

Wireshark：

- 图形化。
- 适合学习协议字段。
- 适合打开 `.pcap` 文件分析。

tcpdump：

- 命令行。
- 适合服务器上快速抓包。
- 适合保存文件后下载分析。

真实工作中经常是：

```text
服务器上用 tcpdump 抓包 -> 保存 pcap -> 本地用 Wireshark 打开。
```

---

## 二、先用 Wireshark 抓 ping

打开 Wireshark，选择当前正在使用的网卡。

如果不确定选哪个，可以观察哪个网卡流量在跳动。

开始抓包后，在终端执行：

```bash
ping 8.8.8.8
```

Wireshark 显示过滤器输入：

```text
icmp
```

你应该看到：

```text
Echo request
Echo reply
```

含义：

- Echo request：你发出的 ping 请求。
- Echo reply：对方返回的 ping 响应。

如果没有看到，可能是：

- 选错网卡。
- ICMP 被拦截。
- 使用的是 WSL2，报文出现在虚拟网卡或 Windows 网卡上。

---

## 三、用 tcpdump 抓 ping

Linux / WSL2：

```bash
sudo tcpdump -i any icmp -nn
```

参数说明：

- `-i any`：监听所有网卡。
- `icmp`：只抓 ICMP。
- `-nn`：不要把 IP 和端口解析成名称。

另一个终端执行：

```bash
ping 8.8.8.8
```

你会看到类似：

```text
IP 192.168.1.10 > 8.8.8.8: ICMP echo request
IP 8.8.8.8 > 192.168.1.10: ICMP echo reply
```

这说明请求和响应都经过了本机。

停止 tcpdump：

```text
Ctrl + C
```

---

## 四、抓 DNS 查询

DNS 通常使用 UDP 53 端口。

tcpdump：

```bash
sudo tcpdump -i any port 53 -nn
```

另一个终端执行：

```bash
dig example.com
```

你应该看到本机向 DNS 服务器发送查询，并收到响应。

Wireshark 显示过滤器：

```text
dns
```

观察：

- 查询的域名。
- 查询类型，例如 A、AAAA。
- 返回的 IP。
- 响应耗时。

---

## 五、抓 HTTP 明文请求

HTTPS 内容是加密的，不适合初学阶段观察 HTTP 头和 body。我们先用本地 HTTP 服务。

如果你还没有服务，可以用 Python 临时启动：

```bash
python3 -m http.server 8080
```

抓包：

```bash
sudo tcpdump -i any tcp port 8080 -nn
```

另一个终端请求：

```bash
curl -v http://127.0.0.1:8080
```

你会看到 TCP 报文。

如果想在 Wireshark 中看 HTTP 内容，可以保存 pcap：

```bash
sudo tcpdump -i any tcp port 8080 -w http-8080.pcap
```

请求一次后停止 tcpdump，再用 Wireshark 打开 `http-8080.pcap`。

显示过滤器：

```text
http
```

---

## 六、为什么抓 127.0.0.1 有时看不到

在不同系统上，回环流量的抓取方式不一样。

Linux 上可以用：

```bash
sudo tcpdump -i lo tcp port 8080 -nn
```

或者：

```bash
sudo tcpdump -i any tcp port 8080 -nn
```

Windows + WSL2 场景可能更复杂。你可以优先抓：

- WSL2 内部请求 WSL2 服务。
- 宿主机局域网 IP 请求服务。
- Docker 容器请求服务。

初学阶段不必被回环抓包细节卡太久，重点是学会抓包思路。

---

## 七、抓包时先问三个问题

每次抓包前先明确：

### 1. 我要抓哪个网卡？

不知道时先用 `any`：

```bash
sudo tcpdump -i any ...
```

### 2. 我要抓哪个协议或端口？

例如：

```bash
icmp
port 53
tcp port 8080
host 8.8.8.8
```

### 3. 我要证明什么？

例如：

```text
证明 DNS 有没有响应。
证明 TCP 有没有建立连接。
证明请求有没有到达服务端。
证明服务端有没有回包。
```

没有问题意识的抓包，很容易抓到一堆看不懂的报文。

---

## 八、常用 tcpdump 命令

抓某个端口：

```bash
sudo tcpdump -i any port 8080 -nn
```

抓某个主机：

```bash
sudo tcpdump -i any host 8.8.8.8 -nn
```

抓某个主机和端口：

```bash
sudo tcpdump -i any host 127.0.0.1 and port 8080 -nn
```

保存到文件：

```bash
sudo tcpdump -i any port 8080 -w capture.pcap
```

读取文件：

```bash
tcpdump -r capture.pcap -nn
```

---

## 九、常见问题

### 1. 抓包看不到任何内容

可能原因：

- 选错网卡。
- 过滤条件太严格。
- 请求根本没有发出。
- 流量走了另一个网络命名空间，例如 Docker 或 WSL2。

### 2. 为什么 HTTPS 看不到 HTTP 内容？

HTTPS 使用 TLS 加密。抓包能看到 TCP 和 TLS 握手，但看不到明文 HTTP Header 和 Body。

初学阶段观察 HTTP 明文，建议使用本地 HTTP 服务。

### 3. tcpdump 输出太多怎么办？

加过滤条件，例如：

```bash
tcp port 8080
host 8.8.8.8
port 53
```

### 4. 抓包文件很大怎么办？

只抓必要端口，或者限制包数量：

```bash
sudo tcpdump -i any port 8080 -c 20 -w capture.pcap
```

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 使用 Wireshark 抓到一次 ping。
- 使用 tcpdump 抓到 ICMP、DNS 或 HTTP 流量。
- 使用 `-i any`、`port`、`host`、`-w` 等基础参数。
- 解释为什么 HTTPS 抓不到明文 HTTP 内容。
- 知道抓包前要明确“抓哪里、抓什么、证明什么”。

完成这些之后，就可以进入下一节：启动一个 Go HTTP 服务并完成端到端访问。

