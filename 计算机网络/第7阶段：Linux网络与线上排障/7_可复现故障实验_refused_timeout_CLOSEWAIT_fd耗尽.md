# 7. 可复现故障实验：refused、timeout、CLOSE_WAIT、fd 耗尽

本节目标：通过本地实验复现几类常见线上网络故障，并学会用命令定位。你会亲手看到 `Connection refused`、`timeout`、`CLOSE_WAIT` 和 `too many open files` 背后的证据。

---

## 一、为什么要做故障实验

线上排障最怕只会背结论：

```text
refused 是端口没开。
timeout 是网络不通。
CLOSE_WAIT 是连接没关。
too many open files 是 fd 不够。
```

这些说法方向没错，但面试和真实工作更看重：

```text
你能不能复现？
你能不能用命令证明？
你能不能定位到代码或配置？
你能不能给出修复方案？
```

本节就是把这些故障做成可操作实验。

---

## 二、实验一：Connection refused

### 1. 确认端口没有监听

```bash
ss -lntp | grep 9090
```

如果没有输出，说明本机 `9090` 没有 TCP 服务监听。

### 2. 访问端口

```bash
curl -v http://127.0.0.1:9090
```

常见输出：

```text
connect to 127.0.0.1 port 9090 failed: Connection refused
```

### 3. 解释

`Connection refused` 通常表示：

```text
目标主机可达，但目标端口没有进程监听，或者被主动拒绝。
```

### 4. 修复

启动一个服务：

```bash
python3 -m http.server 9090
```

另一个终端再访问：

```bash
curl -I http://127.0.0.1:9090
```

如果成功，说明刚才的问题确实是端口没有监听。

---

## 三、实验二：Connection timed out

### 1. 访问一个通常不会响应的地址

```bash
curl -v --connect-timeout 3 http://10.255.255.1:8080
```

可能看到：

```text
Connection timed out
```

### 2. 解释

timeout 通常表示：

```text
连接请求发出后，没有收到有效响应。
```

常见原因：

- 防火墙丢弃。
- 云安全组没放行。
- 路由不通。
- 目标机器无响应。
- 回程路径异常。

### 3. 对比 refused

```text
refused：对方明确拒绝。
timeout：没有等到回应。
```

面试中这句话非常重要。

---

## 四、实验三：监听地址错误

创建 Go 服务：

```bash
mkdir bind-lab
cd bind-lab
go mod init bind-lab
```

`main.go`：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ok")
    })

    log.Println("listen on 127.0.0.1:8080")
    log.Fatal(http.ListenAndServe("127.0.0.1:8080", nil))
}
```

启动：

```bash
go run .
```

查看监听：

```bash
ss -lntp | grep 8080
```

你会看到：

```text
127.0.0.1:8080
```

这表示只监听本机回环地址。

本机访问：

```bash
curl http://127.0.0.1:8080
```

成功。

如果其他机器访问你的局域网 IP：

```text
http://你的局域网IP:8080
```

通常失败。

修复：

```go
http.ListenAndServe(":8080", nil)
```

或：

```go
http.ListenAndServe("0.0.0.0:8080", nil)
```

---

## 五、实验四：制造 CLOSE_WAIT

这个实验用于理解“对方关闭了，本地没关闭”。

创建目录：

```bash
mkdir closewait-lab
cd closewait-lab
go mod init closewait-lab
```

创建 `server.go`：

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func main() {
    ln, err := net.Listen("tcp", ":9091")
    if err != nil {
        panic(err)
    }

    fmt.Println("listen on :9091")

    for {
        conn, err := ln.Accept()
        if err != nil {
            continue
        }

        go func(c net.Conn) {
            // 故意不关闭连接，模拟错误代码
            buf := make([]byte, 1)
            c.Read(buf)
            fmt.Println("client closed or sent data, but server does not close conn")
            time.Sleep(10 * time.Minute)
        }(conn)
    }
}
```

启动：

```bash
go run server.go
```

另一个终端连接后退出：

```bash
nc 127.0.0.1 9091
```

输入任意字符后按 `Ctrl+C`。

查看：

```bash
ss -ant state close-wait | grep 9091
```

可能看到 CLOSE_WAIT。

修复方式：

```go
defer c.Close()
```

读到错误或 EOF 后退出 goroutine 并关闭连接。

---

## 六、实验五：观察 fd 增长

先找到进程 ID：

```bash
ps aux | grep closewait-lab
```

或者：

```bash
lsof -i :9091
```

查看 fd 数：

```bash
ls /proc/进程ID/fd | wc -l
```

多连接几次，再观察 fd 数变化。

如果连接不关闭，fd 会持续占用。

真实线上如果 fd 达到上限，会出现：

```text
too many open files
```

查看限制：

```bash
ulimit -n
cat /proc/进程ID/limits
```

---

## 七、实验六：用 tcpdump 证明请求到没到

抓包：

```bash
sudo tcpdump -i any tcp port 8080 -nn
```

另一个终端请求：

```bash
curl -v http://127.0.0.1:8080
```

如果抓包有 SYN，说明请求到达了本机网络栈。

如果服务日志没有请求，但抓包有 SYN，可能是：

- 端口没监听。
- 连接没建立。
- TLS 没完成。
- 请求被代理层拦截。

如果抓包完全没有包，说明请求可能没有到达机器，要查路由、防火墙、安全组或客户端目标地址。

---

## 八、实验记录模板

建议每次排障都按这个格式记录：

```markdown
# 故障：Connection refused

## 现象

curl 访问 127.0.0.1:9090 返回 refused。

## 证据

ss -lntp | grep 9090 无输出。

## 原因

端口没有服务监听。

## 修复

启动服务或修改访问端口。
```

把故障写成证据链，是从初级到熟练的关键。

---

## 九、常见问题

### 1. CLOSE_WAIT 能不能通过重启解决？

重启能暂时清掉连接，但不能解决代码没有关闭连接的根因。

### 2. timeout 一定是防火墙吗？

不一定，也可能是路由、目标机器、回程路径或目标服务无响应。

### 3. fd 上限调大是否就够了？

不够。调大只能缓解，连接泄漏必须修代码。

### 4. tcpdump 抓不到包是不是一定没请求？

不一定，也可能抓错网卡、过滤条件写错、流量在另一个网络命名空间。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 本地复现 `Connection refused`。
- 本地复现或识别 `Connection timed out`。
- 解释监听 `127.0.0.1` 导致外部访问失败的原因。
- 制造并识别 CLOSE_WAIT。
- 查看进程 fd 数和 fd 限制。
- 用 tcpdump 证明请求是否到达机器。
- 写出一份有现象、证据、原因、修复的排障记录。

