# 1. 安装 Go、Docker 与基础网络工具

本节目标：准备 Go 后端学习计算机网络所需的基础工具，包括 Go、Docker、curl、dig、ss、tcpdump 和 Wireshark，并确认它们可以正常使用。

这一节看起来像环境安装，但它其实是后续所有实验的地基。没有这些工具，后面学习 TCP、HTTP、DNS、排障时只能停留在文字层面。

---

## 一、前置条件

开始之前，建议确认：

- 你能打开一个终端。
- Windows 用户已经安装或准备安装 WSL2 Ubuntu。
- 你有管理员权限安装软件。
- 本机能正常访问互联网。

如果你使用的是 Windows，本教程建议主要在 WSL2 Ubuntu 中执行 Linux 命令，在 Windows 中安装 Docker Desktop 和 Wireshark。

---

## 二、安装并验证 Go

Go 是你后续编写网络服务的主要语言。

安装完成后执行：

```bash
go version
```

如果看到类似输出：

```text
go version go1.22.5 linux/amd64
```

说明 Go 已经可用。

Windows PowerShell 中也可以执行：

```powershell
go version
```

---

## 三、写第一个 Go 程序

创建实验目录：

```bash
mkdir hello-go
cd hello-go
go mod init hello-go
```

创建 `main.go`：

```go
package main

import "fmt"

func main() {
    fmt.Println("hello network")
}
```

运行：

```bash
go run .
```

如果输出：

```text
hello network
```

说明 Go 编译和运行链路正常。

---

## 四、安装 Docker

Docker 用来模拟真实后端部署环境。后续你会用它理解：

- 容器 IP。
- 端口映射。
- 容器内外的 `localhost` 差异。
- 多服务通信。

安装 Docker Desktop 后，执行：

```bash
docker version
```

再运行：

```bash
docker run --rm hello-world
```

如果能看到 Docker 的欢迎信息，说明 Docker 工作正常。

---

## 五、安装 Linux 网络工具

在 Ubuntu / WSL2 中执行：

```bash
sudo apt update
sudo apt install -y curl wget dnsutils iproute2 net-tools tcpdump traceroute lsof netcat-openbsd
```

这些包分别提供：

- `curl`：发 HTTP 请求。
- `dig`：查 DNS。
- `ip`：查看网卡、IP、路由。
- `ss`：查看端口和连接。
- `tcpdump`：抓包。
- `traceroute`：查看路由路径。
- `lsof`：查看端口对应进程。
- `nc`：测试 TCP/UDP 连接。

验证：

```bash
curl --version
dig example.com
ip addr
ss -lnt
tcpdump --version
nc -h
```

注意：`nc -h` 在不同系统中输出可能不同，只要命令存在即可。

---

## 六、Windows PowerShell 替代命令

如果你暂时没有 WSL2，也可以先用 PowerShell 熟悉一些基础命令：

查看 IP：

```powershell
ipconfig
```

查看路由：

```powershell
route print
```

查看端口：

```powershell
netstat -ano
```

测试端口：

```powershell
Test-NetConnection example.com -Port 443
```

查询 DNS：

```powershell
nslookup example.com
```

不过后续生产排障仍建议切到 Linux / WSL2 命令体系。

---

## 七、安装 Wireshark

Wireshark 是图形化抓包工具。安装后先不要急着抓复杂流量，初学阶段只抓三类：

- `icmp`：观察 ping。
- `dns`：观察域名解析。
- `tcp.port == 8080`：观察本地 HTTP 服务访问。

常用显示过滤器：

```text
icmp
dns
tcp
http
tcp.port == 8080
ip.addr == 127.0.0.1
```

注意：Wireshark 的“捕获过滤器”和“显示过滤器”不是一回事。初学阶段先使用显示过滤器即可。

---

## 八、工具安装后的统一检查

请执行下面命令，并把输出记录到自己的笔记里：

```bash
go version
docker version
curl --version
dig example.com
ip addr
ip route
ss -lnt
tcpdump --version
```

如果某个命令失败，不要跳过。后面教程会反复使用这些命令。

---

## 九、常见问题

### 1. WSL2 中执行 docker 命令失败

检查 Docker Desktop 是否开启 WSL integration。

路径通常是：

```text
Docker Desktop -> Settings -> Resources -> WSL Integration
```

开启对应 Ubuntu 发行版后，重启终端。

### 2. tcpdump 提示权限不足

抓包通常需要 root 权限：

```bash
sudo tcpdump -i any
```

### 3. dig 命令不存在

Ubuntu 中 `dig` 属于 `dnsutils`：

```bash
sudo apt install -y dnsutils
```

### 4. 端口被占用

后续启动服务时如果提示端口被占用，可以用：

```bash
ss -lntp | grep 8080
```

查看是谁占用了端口。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 执行 `go version` 并运行一个最小 Go 程序。
- 执行 `docker run --rm hello-world`。
- 使用 `curl`、`dig`、`ip`、`ss`、`tcpdump`。
- 打开 Wireshark 并知道显示过滤器在哪里输入。
- 知道 Windows PowerShell 中有哪些替代命令。

完成这些之后，就可以进入下一节：认识本机网络信息。

