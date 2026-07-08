# 8. 一次 HTTPS 访问失败的四层排障剧本

本节目标：把线上排障拆成一套可以反复使用的剧本，从 DNS、路由、端口、TCP、TLS、HTTP、服务日志逐层定位问题。

很多网络问题看起来都是一句话：

```text
接口访问不了。
```

但背后可能是：

```text
域名解析错。
路由不通。
端口没监听。
安全组拦截。
TCP 建连失败。
TLS 证书错误。
HTTP 返回 502。
业务服务自己报错。
```

本节的目标是让你拿到一个“访问失败”问题时，不再凭感觉乱试，而是按层排查。

---

## 一、准备一个排障记录模板

以后遇到网络故障，先写下这几个字段：

```text
现象：
请求方：
目标域名：
目标端口：
协议：
首次发现时间：
是否所有人都失败：
是否只有某个环境失败：
最近变更：
```

示例：

```text
现象：Go 服务访问 https://api.example.com/users 超时。
请求方：payment-service 容器。
目标域名：api.example.com。
目标端口：443。
协议：HTTPS。
首次发现时间：2026-07-05 10:20。
是否所有人都失败：只有线上 payment-service 失败。
是否只有某个环境失败：本地和测试环境正常。
最近变更：上午 10 点发布过安全组规则。
```

先记录这些信息，是为了避免排障过程中来回切换问题范围。

---

## 二、第一层：确认域名解析

先查 DNS：

```powershell
dig api.example.com
```

如果没有 `dig`，可以用：

```powershell
nslookup api.example.com
```

你要关注：

```text
是否有 A 或 AAAA 记录。
解析到的 IP 是否符合预期。
TTL 是否异常。
是否解析到了内网 IP 或旧 IP。
```

`dig` 输出里重点看：

```text
QUESTION SECTION
ANSWER SECTION
SERVER
Query time
```

如果 `ANSWER SECTION` 为空，说明域名没有解析结果，问题还没到 TCP 层。

如果解析结果和你预期不同，继续检查：

```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts
```

Linux/WSL：

```bash
cat /etc/hosts
cat /etc/resolv.conf
```

容器内也要查：

```powershell
docker exec -it 容器名 sh
cat /etc/resolv.conf
nslookup api.example.com
```

常见结论：

```text
本机解析正常，容器内解析失败：看 Docker DNS 或容器网络。
公司网络解析到内网 IP，外网解析到公网 IP：看 split-horizon DNS。
只有一台机器解析错：看 hosts、本机 DNS 缓存、系统 DNS 配置。
```

---

## 三、第二层：确认路由选择

拿到 IP 后，看系统会从哪里出去：

```bash
ip route get 目标IP
```

Windows：

```powershell
route print
```

Linux 输出示例：

```text
目标IP via 172.20.0.1 dev eth0 src 172.20.5.10
```

你要读出：

```text
via        下一跳网关。
dev        出口网卡。
src        本机使用哪个源 IP。
```

这一步能发现：

- VPN 抢走了路由。
- 多网卡机器选择了错误出口。
- 容器源 IP 不符合安全组白名单。
- 默认网关丢失。

如果路由都不对，后面测端口没有意义。

---

## 四、第三层：确认 TCP 端口能否建立连接

用 `curl` 先看整体：

```powershell
curl.exe -v https://api.example.com/users
```

`curl -v` 输出中重点看：

```text
Trying x.x.x.x:443...
Connected to api.example.com
TLS handshake
HTTP request
HTTP response
```

如果卡在：

```text
Trying x.x.x.x:443...
```

通常是 TCP 建连没有成功。

Linux 可以用：

```bash
nc -vz api.example.com 443
```

或者：

```bash
timeout 3 bash -c '</dev/tcp/api.example.com/443' && echo ok || echo failed
```

结果判断：

```text
Connection refused：目标机器可达，但端口没人监听，或被主动拒绝。
Connection timed out：包可能被防火墙、安全组、路由中间设备丢弃。
No route to host：本机路由或网络不可达。
```

这三种错误的处理方向完全不同。

---

## 五、第四层：抓包确认 SYN 有没有回来

在请求方抓包：

```bash
sudo tcpdump -i any -nn host 目标IP and port 443
```

另一个终端执行：

```bash
curl -v https://api.example.com/users
```

抓包中看 TCP 三次握手：

```text
SYN
SYN, ACK
ACK
```

判断方式：

```text
只看到 SYN 重传：请求发出去了，但没有收到回应。
看到 SYN,ACK 后马上 RST：连接被对端或中间设备重置。
三次握手完成：TCP 层没问题，继续看 TLS/HTTP。
```

这一步非常关键，因为它能把“应用层猜测”变成“网络包证据”。

---

## 六、第五层：检查 TLS

如果 TCP 能连上，但 HTTPS 失败，检查 TLS：

```powershell
openssl s_client -connect api.example.com:443 -servername api.example.com
```

重点看：

```text
Verify return code
subject
issuer
notBefore / notAfter
```

常见问题：

```text
证书过期。
证书域名不匹配。
缺少中间证书。
客户端不信任内部 CA。
SNI 没带或带错。
```

为什么 `-servername` 重要？

因为很多 HTTPS 服务会根据 SNI 返回不同证书。如果不带 SNI，你看到的证书可能不是目标域名真正使用的证书。

---

## 七、第六层：检查 HTTP 状态码

如果 TLS 成功，继续看 HTTP：

```powershell
curl.exe -i https://api.example.com/users
```

常见状态码判断：

```text
200：请求成功。
301/302：重定向，检查 Location。
400：请求格式不符合服务端要求。
401：未认证。
403：无权限或被网关拦截。
404：路径不存在或路由未匹配。
429：被限流。
500：业务服务内部错误。
502：网关连接上游失败。
503：服务不可用或被摘除。
504：网关等待上游超时。
```

后端工程师不能只说“接口挂了”。你要能根据状态码判断问题大致在哪一层。

---

## 八、第七层：对照服务监听状态

如果你能登录目标机器，检查监听：

```bash
ss -lntp
```

查具体端口：

```bash
ss -lntp | grep ':443'
ss -lntp | grep ':8080'
```

你要确认：

```text
端口是否监听。
监听地址是 127.0.0.1 还是 0.0.0.0。
进程名是否符合预期。
PID 是否是最新发布后的进程。
```

典型错误：

```text
服务只监听 127.0.0.1:8080，外部当然访问不到。
容器内监听 8080，但 Docker 没映射端口。
Nginx 监听 443，但 upstream 指向了错误端口。
```

---

## 九、第八层：查看代理和应用日志

Nginx 常看：

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

systemd 服务：

```bash
journalctl -u 服务名 -f
```

Docker：

```bash
docker logs -f 容器名
```

Go 服务建议日志里至少有：

```text
request_id
method
path
status
cost
remote_addr
upstream
error
```

如果日志里完全看不到请求，问题在服务前面：

```text
DNS、路由、防火墙、负载均衡、网关。
```

如果日志里看到请求但返回错误，问题进入应用层：

```text
鉴权、参数、数据库、下游依赖、业务代码。
```

---

## 十、完整排障示例

现象：

```text
payment-service 调用 https://user.internal/api/users/1 超时。
```

排查过程：

1. DNS：

```bash
dig user.internal
```

解析到：

```text
10.20.1.15
```

2. 路由：

```bash
ip route get 10.20.1.15
```

出口是：

```text
dev eth0 src 10.30.2.8
```

3. TCP：

```bash
nc -vz user.internal 443
```

结果：

```text
timed out
```

4. 抓包：

```bash
tcpdump -i any -nn host 10.20.1.15 and port 443
```

只看到 SYN 重传，没有 SYN,ACK。

5. 结论：

```text
请求方已经发包，目标没有回应。
优先检查安全组、防火墙、路由 ACL。
```

6. 最终发现：

```text
上午发布安全组时，只放行了测试网段，没有放行线上 payment-service 网段。
```

修复：

```text
增加 10.30.2.0/24 到 user-service 443 入站规则。
```

验证：

```bash
curl -i https://user.internal/api/users/1
```

返回 `200`。

---

## 十一、常见问题

### 1. 为什么不要一上来重启服务？

重启可能掩盖现场。

如果问题是连接泄漏、CLOSE_WAIT、端口耗尽，重启会暂时恢复，但你会失去定位证据。先采集状态，再决定是否重启。

### 2. `curl` 成功就说明 Go 服务一定成功吗？

不一定。

Go 服务可能使用不同 DNS、不同代理、不同源 IP、不同证书信任库。最好在同一台机器、同一个容器、同一个网络命名空间里测试。

### 3. 为什么抓包看到 SYN 发出去了，还不能证明对端收到了？

因为抓包点在请求方，只能证明包离开了本机协议栈。中间路由、安全组、防火墙都可能丢包。

如果条件允许，可以在目标机器也抓包，确认包是否到达。

### 4. 502 和 504 怎么区分？

常见理解：

```text
502：网关连接上游失败、上游提前断开、协议错误。
504：网关连接上游后，等待上游响应超时。
```

不同代理实现会有细节差异，但这个方向足够指导排障。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 写出 HTTPS 访问失败的排障记录模板。
- 使用 `dig/nslookup` 判断 DNS 是否正常。
- 使用 `ip route get` 判断出口网卡、源 IP 和下一跳。
- 使用 `curl -v` 区分 TCP、TLS、HTTP 阶段。
- 使用 `tcpdump` 判断 SYN、SYN-ACK、RST、重传。
- 使用 `openssl s_client` 检查证书、SNI 和过期时间。
- 使用 `ss -lntp` 判断服务监听地址和端口。
- 根据 502、504、timeout、refused 判断不同故障方向。
- 给出一份有证据链的排障结论。

