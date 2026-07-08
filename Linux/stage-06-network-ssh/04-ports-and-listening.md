# 04：端口与监听

本节目标：理解端口和监听，学会查看本机哪些服务正在监听，能判断端口是否被占用。

本节命令顺序：

```text
1. ss -tunlp
2. ss -tuln
3. ss -tnp
4. netstat -tunlp
5. lsof -i
6. curl localhost
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-06
```

---

## 1. 端口是什么

IP 定位一台主机，端口定位主机上的某个服务。

常见端口：

| 端口 | 服务 |
| --- | --- |
| `22` | SSH |
| `80` | HTTP |
| `443` | HTTPS |
| `3306` | MySQL |
| `5432` | PostgreSQL |
| `6379` | Redis |

如果一个服务要被访问，通常需要：

```text
服务正在运行 -> 监听端口 -> 防火墙允许 -> 网络可达
```

---

## 2. ss：查看监听端口

查看 TCP、UDP 监听端口：

```bash
ss -tunlp
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-t` | TCP |
| `-u` | UDP |
| `-n` | 不解析名称，显示数字端口 |
| `-l` | 只显示监听端口 |
| `-p` | 显示进程信息 |

如果没有权限看到进程名，可以加：

```bash
sudo ss -tunlp
```

---

## 3. 只看监听端口

执行：

```bash
ss -tuln
```

输出中重点看：

| 列 | 含义 |
| --- | --- |
| `Local Address:Port` | 本地监听地址和端口 |
| `Peer Address:Port` | 对端地址和端口 |

常见监听地址：

| 地址 | 含义 |
| --- | --- |
| `127.0.0.1:端口` | 只允许本机访问 |
| `0.0.0.0:端口` | 监听所有 IPv4 地址 |
| `[::]:端口` | 监听 IPv6 所有地址 |

---

## 4. 查看连接状态

查看 TCP 连接：

```bash
ss -tn
```

查看带进程信息的 TCP 连接：

```bash
ss -tnp
```

常见状态：

| 状态 | 含义 |
| --- | --- |
| `LISTEN` | 正在监听 |
| `ESTAB` | 已建立连接 |
| `TIME-WAIT` | 连接关闭后的等待状态 |

---

## 5. netstat：传统查看端口命令

如果没有安装：

```bash
sudo apt install net-tools
```

执行：

```bash
netstat -tunlp
```

`netstat` 比较老，但很多资料和老服务器仍然会用。

新系统优先使用：

```bash
ss -tunlp
```

---

## 6. lsof -i：查看端口被谁占用

查看所有网络相关打开文件：

```bash
sudo lsof -i
```

查看 80 端口：

```bash
sudo lsof -i :80
```

查看 22 端口：

```bash
sudo lsof -i :22
```

---

## 7. 用 Python 临时启动一个 HTTP 服务

进入目录：

```bash
cd ~/linux-practice/stage-06
```

启动临时服务：

```bash
python3 -m http.server 8080
```

另开一个终端，检查端口：

```bash
ss -tunlp | grep 8080
curl -I http://localhost:8080
```

回到第一个终端，按：

```text
Ctrl + C
```

停止服务。

---

## 8. 本节练习

执行：

```bash
ss -tuln
sudo ss -tunlp
netstat -tunlp
sudo lsof -i :22
```

再启动临时 HTTP 服务：

```bash
python3 -m http.server 8080
```

另开终端执行：

```bash
ss -tunlp | grep 8080
curl -I http://localhost:8080
```

---

## 9. 本节小结

必须记住：

```bash
ss -tuln
sudo ss -tunlp
ss -tnp
netstat -tunlp
sudo lsof -i :80
python3 -m http.server 8080
curl -I http://localhost:8080
```

核心思路：

```text
服务访问不了时，要确认服务是否监听了正确端口。
```

完成后进入下一节：`05-ssh-login-keys.md`。

