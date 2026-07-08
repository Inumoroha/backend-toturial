# 5. NAT 原理与 Docker 容器访问外网

本节目标：理解 NAT 如何让私有地址访问公网，并通过 Docker 网络观察容器访问外网的基本路径。

---

## 一、为什么需要 NAT

你的电脑可能是：

```text
192.168.1.10
```

这是私有 IP，不能直接在公网路由。

但你仍然可以访问：

```bash
curl https://example.com
```

原因通常是路由器做了 NAT，把内网源地址转换成公网出口地址。

---

## 二、NAT 基本流程

内网主机访问公网：

```text
192.168.1.10:50000 -> 93.184.216.34:443
```

经过路由器后可能变成：

```text
公网IP:62000 -> 93.184.216.34:443
```

路由器记录映射关系：

```text
公网IP:62000 -> 192.168.1.10:50000
```

服务器响应回来时，路由器根据映射转回内网机器。

---

## 三、查看公网出口 IP

```bash
curl ifconfig.me
```

再查看本机 IP：

```bash
ip addr
```

如果两者不同，说明你大概率在 NAT 后面。

---

## 四、Docker bridge 与 NAT

Docker 默认 bridge 网络中，容器通常获得类似：

```text
172.17.0.2
```

容器访问外网时，宿主机通常会通过 NAT 转换源地址。

查看 Docker 网络：

```bash
docker network inspect bridge
```

进入容器：

```bash
docker run --rm -it alpine sh
```

容器内：

```sh
apk add --no-cache curl iproute2
ip addr
ip route
curl https://example.com
```

观察：

- 容器 IP。
- 默认网关。
- 是否能访问外网。

---

## 五、端口映射也是 NAT 思想

运行：

```bash
docker run --rm -d --name web -p 8080:80 nginx:alpine
```

访问：

```bash
curl http://127.0.0.1:8080
```

这里发生的是：

```text
宿主机 8080 -> 容器 80
```

这和 NAT/端口转发思想类似。

---

## 六、后端中的 NAT 影响

### 1. 访问日志中的客户端 IP 不一定真实

经过 NAT、代理、负载均衡后，服务看到的 `RemoteAddr` 可能是代理或网关地址。

需要配合：

```text
X-Forwarded-For
X-Real-IP
```

但这些 Header 只能信任可信代理设置的值。

### 2. 服务注册不能乱写 NAT 后地址

容器内地址、Pod 地址、宿主机地址、公网地址可能都不同。服务发现必须使用调用方可达的地址。

### 3. 短连接过多可能耗尽 NAT 映射或临时端口

大量短连接会给 NAT 设备、客户端临时端口和服务端连接状态带来压力。

---

## 补充实验：观察容器访问外网时的源地址变化

进入容器：

```bash
docker run --rm -it alpine sh
```

在容器里查看 IP：

```sh
ip addr
ip route
```

访问外网：

```sh
wget -O- https://ifconfig.me
```

再在宿主机执行：

```bash
curl https://ifconfig.me
```

你通常会看到容器和宿主机对外展示的是同一个公网出口地址。这说明容器的私有 IP 在出宿主机时被 NAT 成了宿主机的出口地址。

---

## 补充排障：容器能 ping IP 但不能访问域名

如果容器里：

```sh
ping -c 3 8.8.8.8
```

成功，但：

```sh
ping -c 3 example.com
```

失败，说明 IP 层和 NAT 可能正常，DNS 有问题。

检查：

```sh
cat /etc/resolv.conf
nslookup example.com
```

排障结论可以写：

```text
容器默认路由存在。
访问公网 IP 成功。
域名解析失败。
问题优先定位在容器 DNS 配置或 Docker DNS。
```

这样就不会把 DNS 问题误判成 NAT 问题。

---

## 七、常见问题

### 1. NAT 会不会改变目标地址？

源 NAT 通常改变源地址。端口映射或目标 NAT 可能改变目标地址。

### 2. 多台内网机器如何共享一个公网 IP？

NAT 通过端口映射区分不同连接。

### 3. Docker 容器为什么能访问外网？

默认 bridge 网络下，宿主机通常为容器流量做 NAT。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 NAT 的基本流程。
- 使用 `curl ifconfig.me` 对比本机内网 IP 和公网出口 IP。
- 查看 Docker bridge 网络。
- 解释 Docker 容器访问外网为什么通常依赖宿主机 NAT。
- 解释端口映射 `8080:80` 的含义。
