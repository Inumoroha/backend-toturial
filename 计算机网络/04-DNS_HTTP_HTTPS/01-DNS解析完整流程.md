# 01-DNS 解析完整流程

## 学习目标

理解域名如何变成 IP 地址，掌握 `dig` 的基本用法，并知道 DNS 慢、DNS 失败会如何影响 Go 后端服务。

## 一、DNS 解决什么问题

人更容易记住域名，例如：

```text
api.example.com
```

机器通信需要 IP 地址。DNS 的作用就是把域名解析成 IP 地址，或查询其他类型的记录。

## 二、常见 DNS 记录

| 记录 | 作用 |
| --- | --- |
| A | 域名到 IPv4 地址 |
| AAAA | 域名到 IPv6 地址 |
| CNAME | 域名别名 |
| MX | 邮件服务器 |
| TXT | 文本记录，常用于验证 |
| NS | 域名由哪些权威 DNS 服务器负责 |

后端工程师最常接触 A、AAAA、CNAME。

## 三、DNS 解析流程

以 `www.example.com` 为例：

1. 应用先查本机缓存。
2. 查操作系统 hosts 文件。
3. 请求本地配置的递归 DNS 服务器。
4. 递归 DNS 从根域名服务器开始查询。
5. 查询 `.com` 顶级域服务器。
6. 查询 `example.com` 权威服务器。
7. 得到 `www.example.com` 的记录。
8. 结果按 TTL 缓存。

通常应用不会直接访问根域名服务器，而是把请求交给递归 DNS。

## 四、TTL 与缓存

TTL 表示 DNS 记录可以缓存多久。

TTL 太短：

- 变更生效快。
- DNS 查询压力大。

TTL 太长：

- 查询少。
- 变更生效慢。

服务迁移、域名切流时必须考虑 TTL。

## 五、实验：dig 基础

查询 A 记录：

```bash
dig example.com
```

只看简洁结果：

```bash
dig +short example.com
```

查询 CNAME：

```bash
dig www.example.com CNAME
```

追踪完整解析链路：

```bash
dig +trace example.com
```

## 六、Go 中的 DNS

Go 可以使用纯 Go resolver，也可以通过 cgo 调用系统 resolver。一般你不需要强行记住所有细节，但要知道：

- DNS 解析可能产生耗时。
- DNS 失败会导致连接建立前就失败。
- 容器环境中的 `/etc/resolv.conf` 会影响 Go 程序。
- Kubernetes 中服务名解析依赖集群 DNS。

示例：

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    ips, err := net.LookupHost("example.com")
    if err != nil {
        panic(err)
    }
    fmt.Println(ips)
}
```

## 七、DNS 问题排查

常见现象：

- 请求偶发慢，但服务端没收到请求。
- 只有容器内解析失败。
- 本机能访问，服务器不能访问。
- 域名切换后部分机器仍访问旧 IP。

排查命令：

```bash
cat /etc/resolv.conf
dig api.example.com
dig @8.8.8.8 api.example.com
curl -v https://api.example.com
```

## 八、Go 后端关联

HTTP Client 发请求时，DNS 在 TCP 连接之前发生。

如果 DNS 慢，你看到的可能是：

- 请求总耗时长。
- 服务端没有日志。
- tcpdump 中没有对目标 IP 的 TCP 握手。

所以排查时要区分：

```text
DNS 慢
TCP 连接慢
TLS 握手慢
服务端处理慢
响应体传输慢
```

## 九、练习题

1. A 记录和 CNAME 记录有什么区别？
2. TTL 太短或太长分别有什么问题？
3. 为什么域名解析失败时服务端可能完全没有日志？
4. 如何指定 DNS 服务器查询一个域名？

## 十、验收标准

你能用 `dig` 查询域名记录，能解释递归 DNS 和权威 DNS 的关系，也能把 DNS 纳入后端请求慢的排查链路。

