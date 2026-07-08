# 02-ARP、ICMP 与 NAT 实验教程

## 学习目标

掌握 ARP、ICMP、NAT 的作用，并通过命令和抓包观察它们。它们经常出现在“能不能连通”“为什么访问不到”“容器为什么能访问外网”等问题中。

## 一、ARP 解决什么问题

ARP 用于在局域网内把 IP 地址解析为 MAC 地址。

主机 A 想发包给同网段主机 B：

1. A 知道 B 的 IP。
2. A 不知道 B 的 MAC。
3. A 广播 ARP 请求：谁是这个 IP？
4. B 回复自己的 MAC。
5. A 缓存这个映射。

查看 ARP 缓存：

```bash
arp -a
ip neigh
```

## 二、实验：观察 ARP

1. 查看网关 IP：

   ```bash
   ip route
   ```

2. 清理或等待 ARP 缓存过期后，ping 网关：

   ```bash
   ping 网关IP
   ```

3. 查看 ARP 缓存：

   ```bash
   ip neigh
   ```

4. 用 Wireshark 过滤：

   ```text
   arp
   ```

你会看到 ARP Request 和 ARP Reply。

## 三、ICMP 解决什么问题

ICMP 用于传递网络层控制消息。我们最熟悉的是 `ping`，它使用 ICMP Echo Request 和 Echo Reply。

`ping` 能证明：

- 目标 IP 可达。
- 往返延迟大致是多少。
- 是否有明显丢包。

`ping` 不能证明：

- TCP 端口一定开放。
- HTTP 服务一定正常。
- 防火墙没有限制 TCP。

所以排查服务时，`ping` 成功只是第一步。

## 四、实验：ping 与端口访问区别

```bash
ping example.com
curl -v https://example.com
```

可能出现：

- `ping` 失败，但 HTTP 成功：目标禁用了 ICMP。
- `ping` 成功，但 HTTP 失败：端口、防火墙、服务本身有问题。

## 五、NAT 解决什么问题

NAT 允许内网机器通过一个公网 IP 访问外部网络。

家庭网络中常见流程：

1. 你的电脑 IP 是 `192.168.1.10`。
2. 你访问公网服务器。
3. 路由器把源地址改成公网 IP，并记录映射。
4. 服务器响应公网 IP。
5. 路由器根据映射把响应转回你的电脑。

Docker 默认 bridge 网络也会使用类似 NAT 机制，让容器访问外网。

## 六、实验：Docker NAT 观察

启动容器：

```bash
docker run --rm -it alpine sh
```

容器中执行：

```sh
ip addr
ip route
ping 8.8.8.8
```

在宿主机查看 Docker 网络：

```bash
docker network ls
docker network inspect bridge
```

观察：

- 容器 IP 通常是私有地址。
- 默认网关通常是 Docker bridge。
- 容器访问外网时通过宿主机 NAT。

## 七、Go 后端关联

NAT 和容器网络会影响服务部署：

- 容器内监听 `8080`，宿主机不一定能直接访问，除非做端口映射。
- 服务注册时不能随便注册容器内 IP。
- 客户端看到的远端地址可能是网关或代理地址。
- 访问日志里的源 IP 可能被 NAT 或反向代理改写。

## 八、练习题

1. ARP 为什么只在局域网内工作？
2. `ping` 成功是否说明 HTTP 服务一定正常？
3. NAT 为什么能让多个内网设备共享公网 IP？
4. Docker 容器访问外网时通常经过什么机制？

## 九、验收标准

你能用 `ip neigh` 或 `arp -a` 查看 ARP 缓存，能解释 `ping` 的局限，也能说明 Docker 容器访问外网为什么通常需要 NAT。

