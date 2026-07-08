# 1. OSI 七层与 TCP/IP 四层模型

本节目标：理解网络分层的意义，掌握 OSI 七层模型和 TCP/IP 四层模型的对应关系，并能用分层思想分析一次 HTTP 请求。

网络分层不是为了考试背诵，而是为了把复杂问题拆开。后端排障时，如果你能判断问题发生在 DNS、TCP、TLS、HTTP、路由还是防火墙层面，排查效率会完全不同。

---

## 一、为什么网络要分层

一次网络请求看似简单：

```bash
curl http://127.0.0.1:8080/hello
```

背后却涉及很多问题：

- 应用如何表达请求？这是 HTTP 要解决的。
- 数据如何可靠送达？这是 TCP 要解决的。
- 数据要发给哪台主机？这是 IP 要解决的。
- 同一局域网里下一跳是谁？这是以太网和 ARP 要解决的。
- 比特如何通过网线或无线传输？这是物理层要解决的。

如果所有能力混在一起，协议会非常复杂。分层的好处是：

- 每层只关心自己的职责。
- 上层可以复用下层能力。
- 某一层变化时，不一定影响所有层。
- 排障时可以逐层定位。

---

## 二、OSI 七层模型

OSI 七层模型如下：

| 层级 | 名称 | 解决的问题 | 常见例子 |
| --- | --- | --- | --- |
| 7 | 应用层 | 应用如何表达数据 | HTTP、DNS、SMTP |
| 6 | 表示层 | 数据格式、编码、加密 | JSON、TLS 的部分能力 |
| 5 | 会话层 | 会话建立、保持、恢复 | 登录态、RPC 会话概念 |
| 4 | 传输层 | 进程到进程通信 | TCP、UDP |
| 3 | 网络层 | 主机到主机寻址和路由 | IP、ICMP |
| 2 | 数据链路层 | 同一链路内传输 | Ethernet、Wi-Fi、ARP |
| 1 | 物理层 | 比特如何在介质上传输 | 网线、光纤、无线电 |

初学时不要纠结某个协议严格属于哪一层。模型的价值在于帮助你组织理解。

---

## 三、TCP/IP 四层模型

工程中更常用 TCP/IP 四层模型：

| TCP/IP 层级 | 大致对应 OSI | 常见协议 |
| --- | --- | --- |
| 应用层 | OSI 5-7 层 | HTTP、DNS、TLS、gRPC |
| 传输层 | OSI 4 层 | TCP、UDP |
| 网络层 | OSI 3 层 | IP、ICMP |
| 网络接口层 | OSI 1-2 层 | Ethernet、Wi-Fi、ARP |

Go 后端开发最常写的是应用层代码，例如：

```go
http.HandleFunc("/hello", handler)
```

但这个 HTTP 请求最终要依赖：

```text
HTTP -> TCP -> IP -> Ethernet/Wi-Fi
```

---

## 四、一次 HTTP 请求如何分层

假设你访问：

```bash
curl http://example.com:80/index.html
```

大致流程：

```text
应用层：构造 HTTP GET /index.html 请求。
传输层：通过 TCP 连接 example.com 的 80 端口。
网络层：通过 IP 把包送到目标主机。
网络接口层：在每一跳通过 MAC 地址交给下一台设备。
```

注意：

```text
端口是传输层概念。
IP 是网络层概念。
MAC 是链路层概念。
HTTP 路径是应用层概念。
```

这几个概念不要混用。

---

## 五、封装与解封装

发送数据时，上层数据会被下层加头部。

例如 HTTP 请求发送前会变成：

```text
HTTP 数据
TCP 头 + HTTP 数据
IP 头 + TCP 头 + HTTP 数据
以太网头 + IP 头 + TCP 头 + HTTP 数据
```

这个过程叫封装。

接收方收到后会反过来拆：

```text
以太网帧 -> IP 包 -> TCP 段 -> HTTP 数据
```

这个过程叫解封装。

---

## 六、用 curl 观察分层线索

执行：

```bash
curl -v http://example.com
```

你可能看到：

```text
Trying 93.184.216.34:80...
Connected to example.com
> GET / HTTP/1.1
> Host: example.com
< HTTP/1.1 200 OK
```

这里能看到：

- `Trying IP:80`：DNS 之后开始 TCP 连接。
- `Connected`：TCP 连接建立成功。
- `GET / HTTP/1.1`：应用层 HTTP 请求。
- `HTTP/1.1 200 OK`：应用层 HTTP 响应。

`curl -v` 看不到所有链路层细节，但能帮助你区分请求卡在哪个阶段。

---

## 七、用 tcpdump 观察 TCP/IP

抓 HTTP 请求：

```bash
sudo tcpdump -i any host example.com -nn
```

另一个终端执行：

```bash
curl http://example.com
```

你会看到 IP 和 TCP 报文。

如果想看 HTTP 内容，可以保存后用 Wireshark 打开：

```bash
sudo tcpdump -i any host example.com -w example-http.pcap
```

注意：如果访问的是 HTTPS，你能看到 TCP 和 TLS，但看不到 HTTP 明文内容。

---

## 八、Go 后端中的分层

当你写：

```go
resp, err := http.Get("https://example.com")
```

你操作的是应用层 API。

但底层会发生：

```text
DNS 解析。
TCP 连接。
TLS 握手。
HTTP 请求。
HTTP 响应。
连接复用或关闭。
```

当你写：

```go
conn, err := net.Dial("tcp", "example.com:80")
```

你直接操作的是传输层 TCP 连接。

当你排查：

```bash
ip route
```

你观察的是网络层路由。

---

## 九、常见误区

### 1. HTTP 是不是传输层协议？

不是。HTTP 是应用层协议，通常运行在 TCP 或 QUIC 之上。

### 2. TCP 是否负责找到目标机器？

不负责。TCP 负责进程到进程的可靠传输，目标主机寻址主要由 IP 负责。

### 3. IP 是否保证可靠传输？

不保证。IP 是尽力而为，可能丢包、乱序。可靠性主要由 TCP 或应用层机制补充。

### 4. 分层是不是严格不能跨层？

模型是帮助理解的，真实协议实现可能有交叉。例如 TLS 通常放在应用和传输之间理解。

---

## 十、常见问题

### 1. OSI 七层是不是必须逐层背下来？

不需要死背。你更应该能用分层思想定位问题，例如 DNS、TCP、TLS、HTTP 分别属于不同层次。

### 2. TLS 到底算哪一层？

教学模型里常放在表示层或应用层附近理解。工程排障中更重要的是知道它发生在 TCP 建立之后、HTTP 明文传输之前。

### 3. Go 写 HTTP 服务时是不是只关心应用层？

代码主要在应用层，但超时、连接池、监听地址、端口、路由都会影响服务可用性。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 说出 OSI 七层模型每层大致职责。
- 说出 TCP/IP 四层模型以及常见协议。
- 解释 HTTP、TCP、IP、MAC 分别处在哪一层。
- 画出一次 HTTP 请求的封装过程。
- 使用 `curl -v` 判断请求至少经过了 TCP 和 HTTP 阶段。

掌握这些之后，就可以进入下一节：MAC 地址、IP 地址、端口号分别解决什么问题。
