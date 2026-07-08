# 7. 综合实验：路由、ARP、ICMP、NAT 全链路观察

本节目标：把本阶段所有知识点串成一个完整实验。你会从本机 IP 开始，依次观察路由、网关、ARP、ICMP、Docker NAT 和端口映射，最后形成一套能用于真实排障的网络层检查流程。

前面几节分别讲了 IP、路由、ARP、ICMP、NAT。本节不再只解释概念，而是要求你动手证明：

```text
我的机器是谁？
默认网关是谁？
访问外网走哪条路？
下一跳 MAC 是谁？
ping 能证明什么？
容器为什么能访问外网？
端口映射到底映射了什么？
```

---

## 一、实验准备

建议在 Linux 或 WSL2 Ubuntu 中完成。

需要工具：

```bash
ip
dig
curl
ping
traceroute
tcpdump
docker
```

如果缺少工具：

```bash
sudo apt update
sudo apt install -y dnsutils iproute2 curl traceroute tcpdump net-tools
```

验证 Docker：

```bash
docker version
docker run --rm hello-world
```

---

## 二、第一步：确认本机网络身份

执行：

```bash
ip addr
```

找到主要网卡，例如 `eth0`，记录：

```text
网卡名：
IPv4：
CIDR：
是否私有地址：
```

示例：

```text
eth0
192.168.1.10/24
私有地址
```

为什么要做这一步？

因为所有排障都要先知道“请求从哪里发出”。如果你连源 IP、网卡、网段都不知道，后面看到路由和抓包就很难判断。

---

## 三、第二步：确认默认网关

执行：

```bash
ip route
```

你可能看到：

```text
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

记录：

```text
默认网关：
本地直连网段：
默认出口网卡：
```

解释：

- `192.168.1.0/24` 是本地直连网段。
- 访问本地网段不用经过默认网关。
- 访问其他网段，例如 `8.8.8.8`，会交给默认网关。

---

## 四、第三步：判断访问公网 IP 的下一跳

执行：

```bash
ip route get 8.8.8.8
```

示例输出：

```text
8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.10
```

逐段解释：

- `8.8.8.8`：目标 IP。
- `via 192.168.1.1`：下一跳网关。
- `dev eth0`：从 `eth0` 发出。
- `src 192.168.1.10`：源 IP。

如果没有 `via`，可能说明目标在直连网段。  
如果提示没有路由，说明本机不知道怎么到目标地址。

---

## 五、第四步：观察 ARP 邻居缓存

先 ping 默认网关：

```bash
ping -c 2 默认网关IP
```

再查看邻居缓存：

```bash
ip neigh
```

你可能看到：

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

这说明本机已经知道默认网关的 MAC 地址。

如果访问 `8.8.8.8`，本机不需要知道 `8.8.8.8` 的 MAC，只需要知道默认网关的 MAC。因为当前这一跳是：

```text
本机 -> 默认网关
```

---

## 六、第五步：抓 ARP 报文

打开一个终端抓包：

```bash
sudo tcpdump -i any arp -nn
```

另一个终端访问一个局域网地址或 ping 网关：

```bash
ping -c 1 默认网关IP
```

你可能看到：

```text
ARP, Request who-has 192.168.1.1 tell 192.168.1.10
ARP, Reply 192.168.1.1 is-at aa:bb:cc:dd:ee:ff
```

如果看不到，可能是 ARP 缓存已经存在。可以等待缓存变化，或者换一个同网段内未访问过的 IP。

不要为了清缓存随意在生产机器上操作。本地学习可以观察即可。

---

## 七、第六步：用 ICMP 判断网络层可达

执行：

```bash
ping -c 4 8.8.8.8
```

观察：

- 是否有响应。
- 延迟是多少。
- 是否丢包。

再执行：

```bash
ping -c 4 example.com
```

如果 `8.8.8.8` 能 ping 通，但 `example.com` 不行，可能是 DNS 问题。  
如果 `example.com` 能解析但 ping 不通，也可能是目标禁用了 ICMP。

结论：

```text
ping 是网络层线索，不是应用可用性的最终证明。
```

---

## 八、第七步：用 traceroute 观察路径

执行：

```bash
traceroute example.com
```

如果没有安装：

```bash
sudo apt install -y traceroute
```

你可能看到多个跳点，也可能看到 `*`。

解释：

- 每一行大致代表一跳。
- `*` 不一定表示业务不通，可能只是中间设备不回复探测报文。
- traceroute 更适合提供路径线索，而不是做唯一判断。

---

## 九、第八步：观察 Docker bridge 网络

查看 Docker 网络：

```bash
docker network inspect bridge
```

重点看：

```text
Subnet
Gateway
Containers
```

启动容器：

```bash
docker run --rm -it alpine sh
```

容器内安装工具：

```sh
apk add --no-cache iproute2 curl
```

查看容器网络：

```sh
ip addr
ip route
```

你可能看到：

```text
容器 IP：172.17.0.2
默认网关：172.17.0.1
```

容器访问外网：

```sh
curl -I https://example.com
```

这通常依赖宿主机 NAT。

---

## 十、第九步：观察 Docker 端口映射

启动 Nginx：

```bash
docker run --rm -d --name net-nginx -p 8080:80 nginx:alpine
```

访问：

```bash
curl -I http://127.0.0.1:8080
```

查看端口映射：

```bash
docker port net-nginx
```

输出类似：

```text
80/tcp -> 0.0.0.0:8080
```

解释：

```text
宿主机 8080 -> 容器 80
```

查看宿主机监听：

```bash
ss -lntp | grep 8080
```

清理：

```bash
docker stop net-nginx
```

---

## 十一、故障模拟一：访问错误端口

确保没有服务监听 9090：

```bash
ss -lntp | grep 9090
```

访问：

```bash
curl -v http://127.0.0.1:9090
```

常见现象：

```text
Connection refused
```

解释：

```text
目标主机可达，但目标端口没有程序监听。
```

---

## 十二、故障模拟二：容器未发布端口

启动 Nginx，但不加 `-p`：

```bash
docker run --rm -d --name no-port-nginx nginx:alpine
```

访问宿主机 8080：

```bash
curl -v http://127.0.0.1:8080
```

如果没有其他服务监听，通常失败。

查看：

```bash
docker port no-port-nginx
```

不会看到端口映射。

解释：

```text
容器内 80 端口有服务，不代表宿主机 8080 自动可访问。
```

清理：

```bash
docker stop no-port-nginx
```

---

## 十三、完整排障模板

当一个服务访问失败时，可以按下面顺序写入自己的笔记：

```text
1. 域名：
   dig +short 域名

2. 目标 IP：
   记录解析结果

3. 路由：
   ip route get 目标IP

4. 网关/邻居：
   ip neigh

5. ICMP：
   ping 目标IP

6. TCP 端口：
   nc -vz 目标IP 端口

7. HTTP：
   curl -v URL

8. 抓包：
   sudo tcpdump -i any host 目标IP -nn
```

这样回答面试排障题时会非常清晰。

---

## 十四、常见问题

### 1. 为什么同网段访问要看 ARP？

因为同网段内真正发送以太网帧时需要目标 MAC，ARP 就是把 IP 解析成 MAC 的协议。

### 2. Docker 容器能访问外网是否说明端口映射没问题？

不能。容器访问外网是出方向 NAT，宿主机访问容器服务需要端口映射，是另一个方向的问题。

### 3. ping 失败是否要立刻认为网络断了？

不一定。ICMP 可能被禁。要继续测试 TCP 端口和 HTTP。

### 4. traceroute 有星号是否说明那一跳坏了？

不一定。很多设备不回复 traceroute 探测，但仍然转发业务流量。

---

## 十五、本节达标标准

学完本节后，你应该能够做到：

- 从 `ip addr` 判断本机网络身份。
- 从 `ip route` 找到默认网关。
- 使用 `ip route get` 判断访问目标的下一跳。
- 使用 `ip neigh` 解释 ARP 缓存。
- 使用 `ping` 和 `traceroute` 获取网络层线索。
- 使用 Docker 验证容器 IP、默认网关、NAT 和端口映射。
- 给出一套从域名到目标端口的网络层排障流程。

