# 01-Linux 网络命令排障

## 学习目标

掌握后端工程师最常用的 Linux 网络排障命令，并形成从 DNS、连通性、端口、连接状态到抓包的排查顺序。

## 一、排障总思路

遇到网络问题时，不要一开始就猜代码问题。建议按层定位：

1. 域名是否能解析？
2. 目标 IP 是否可达？
3. 路由是否正确？
4. 目标端口是否开放？
5. TCP 连接是否建立？
6. TLS 是否握手成功？
7. HTTP 状态码是什么？
8. 服务端是否收到请求？
9. 应用日志和耗时在哪个阶段异常？

## 二、DNS 排查

```bash
dig api.example.com
dig +short api.example.com
dig @8.8.8.8 api.example.com
cat /etc/resolv.conf
```

判断：

- 是否解析成功。
- 解析到的 IP 是否符合预期。
- 使用不同 DNS 服务器结果是否不同。
- 容器内 `/etc/resolv.conf` 是否异常。

## 三、连通性排查

```bash
ping 目标IP
traceroute 目标IP
ip route get 目标IP
```

注意：

- `ping` 失败不一定代表 TCP 不通。
- 有些云厂商或服务器会禁 ICMP。
- `ip route get` 能快速看到访问目标走哪个网卡和网关。

## 四、端口排查

查看监听端口：

```bash
ss -lntp
```

查看指定端口：

```bash
ss -lntp | grep 8080
lsof -i :8080
```

测试端口：

```bash
nc -vz 127.0.0.1 8080
curl -v http://127.0.0.1:8080
```

常见结论：

- `Connection refused`：目标主机可达，但端口没人监听或被拒绝。
- `Connection timed out`：网络不通、防火墙丢弃、路由问题。
- `No route to host`：路由不可达或网络隔离。

## 五、连接状态排查

查看所有 TCP 连接：

```bash
ss -antp
```

按状态查看：

```bash
ss -ant state established
ss -ant state time-wait
ss -ant state close-wait
```

关注：

- `ESTABLISHED` 是否过多。
- `TIME_WAIT` 是否大量增长。
- `CLOSE_WAIT` 是否持续堆积。
- 某个远端地址连接是否异常。

## 六、抓包排查

按端口抓包：

```bash
sudo tcpdump -i any port 8080 -nn
```

按主机抓包：

```bash
sudo tcpdump -i any host 10.0.0.5 -nn
```

写入文件，用 Wireshark 分析：

```bash
sudo tcpdump -i any port 8080 -w traffic.pcap
```

抓包能证明：

- 请求有没有发出去。
- 对方有没有回包。
- TCP 握手是否完成。
- 是否有 RST。
- 是否有重传。

## 七、系统限制排查

文件描述符：

```bash
ulimit -n
cat /proc/$(pidof your-app)/limits
```

临时端口范围：

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

连接队列相关参数：

```bash
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
```

## 八、常见问题速查

| 现象 | 优先排查 |
| --- | --- |
| 域名无法访问 | `dig`、`/etc/resolv.conf` |
| IP 可 ping，端口不通 | `ss -lntp`、防火墙、安全组 |
| curl 超时 | 路由、防火墙、服务慢、抓包 |
| 502 | 网关到上游连接失败 |
| 504 | 网关等上游响应超时 |
| CLOSE_WAIT 多 | 应用未关闭连接 |
| too many open files | fd 上限、连接泄漏 |

## 九、练习题

1. `Connection refused` 和 `Connection timed out` 区别是什么？
2. 如何确认服务是否真的监听了 `8080` 端口？
3. 如何证明请求没有到达服务端？
4. 大量 CLOSE_WAIT 优先怀疑什么？

## 十、验收标准

你能拿到一个“接口访问失败”的问题后，用命令逐层排查并给出证据，而不是只凭感觉猜原因。

