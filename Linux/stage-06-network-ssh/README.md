# 第六阶段：网络基础与远程管理教程

本阶段目标：理解 Linux 网络排查的基本路径，能查看 IP、路由、DNS、端口，能使用 `curl`/`wget` 测试 HTTP，能通过 SSH 远程登录，并使用 `scp`/`rsync` 传输文件。

建议学习时间：1 周。  
建议学习方式：按文件顺序学习，每节先看“本节命令顺序”，再跟着练习。  
练习目录：`~/linux-practice/stage-06`

---

## 学习文件顺序

| 顺序 | 文件 | 主题 | 重点命令 |
| --- | --- | --- | --- |
| 1 | `01-ip-and-route.md` | IP、网卡与路由 | `ip addr`、`ip route`、`hostname -I` |
| 2 | `02-connectivity-and-dns.md` | 连通性与 DNS | `ping`、`traceroute`、`dig`、`nslookup` |
| 3 | `03-http-curl-wget.md` | HTTP 请求测试 | `curl`、`wget` |
| 4 | `04-ports-and-listening.md` | 端口与监听 | `ss`、`netstat`、`lsof` |
| 5 | `05-ssh-login-keys.md` | SSH 登录与密钥 | `ssh`、`ssh-keygen`、`ssh-copy-id` |
| 6 | `06-scp-rsync.md` | 文件传输 | `scp`、`rsync` |
| 7 | `07-firewall-ufw.md` | 防火墙基础 | `ufw` |
| 8 | `08-stage-practice.md` | 第六阶段综合练习 | 网络排查、SSH、文件传输、端口检查 |

---

## 第六阶段命令学习总顺序

建议按下面顺序学习：

```text
1. ip addr
2. ip route
3. hostname -I
4. ping
5. traceroute
6. dig
7. nslookup
8. curl
9. wget
10. ss
11. netstat
12. lsof
13. ssh
14. ssh-keygen
15. ssh-copy-id
16. scp
17. rsync
18. ufw
```

---

## 开始前准备

创建第六阶段练习目录：

```bash
mkdir -p ~/linux-practice/stage-06/{data,downloads,logs,output,ssh-demo,tmp}
cd ~/linux-practice/stage-06
pwd
```

安装常用网络工具：

```bash
sudo apt update
sudo apt install curl wget dnsutils traceroute net-tools lsof rsync openssh-client
```

可选：如果你要在本机或虚拟机上练习 SSH 服务端：

```bash
sudo apt install openssh-server
```

说明：

- WSL 中部分网络和 systemd 行为可能和真实服务器不同。
- SSH 服务、防火墙、端口开放更推荐在虚拟机或云服务器中练习。
- 在云服务器上修改防火墙前，务必先确认 SSH 端口不会被挡掉。

---

## 本阶段网络排查思路

遇到“访问不了”的问题，可以按这个顺序排查：

```text
1. 本机有没有 IP？
2. 默认路由是否存在？
3. 能否 ping 通网关？
4. 能否 ping 通公网 IP？
5. DNS 能否解析域名？
6. 目标端口是否开放？
7. 本机服务是否监听？
8. 防火墙是否拦截？
9. 应用日志里有没有错误？
```

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 能查看本机 IP 地址和默认路由。
- 能用 `ping` 判断基础连通性。
- 能用 `dig` 或 `nslookup` 判断 DNS 是否正常。
- 能用 `curl` 查看 HTTP 响应头和响应内容。
- 能用 `ss` 查看本机监听端口。
- 能使用 SSH 登录远程 Linux。
- 能创建 SSH 密钥并理解公钥、私钥的区别。
- 能用 `scp` 复制文件。
- 能用 `rsync` 同步目录。
- 能理解 `ufw` 的基本允许、拒绝和状态查看。

