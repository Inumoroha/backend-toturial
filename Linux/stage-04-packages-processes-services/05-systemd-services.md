# 05：systemd 服务管理

本节目标：学会使用 `systemctl` 查看、启动、停止、重启服务，并设置开机自启。

本节命令顺序：

```text
1. systemctl status
2. systemctl start
3. systemctl stop
4. systemctl restart
5. systemctl reload
6. systemctl enable
7. systemctl disable
8. systemctl is-active
9. systemctl is-enabled
10. systemctl list-units
```

说明：WSL 环境中 systemd 可能未启用。如果 `systemctl` 报错，可以优先在虚拟机 Ubuntu Server 中练习本节。

---

## 1. systemd 是什么

`systemd` 是很多 Linux 发行版默认的系统和服务管理器。

它负责：

- 启动系统服务。
- 停止服务。
- 管理开机自启。
- 记录服务状态。
- 和日志系统配合查看服务日志。

管理命令是：

```bash
systemctl
```

---

## 2. 安装一个练习服务：Nginx

如果还没安装：

```bash
sudo apt update
sudo apt install nginx
```

如果你不想安装 Nginx，也可以用系统已有服务练习，例如：

```bash
systemctl status ssh
```

服务名因系统而异，Ubuntu 上 SSH 服务通常叫 `ssh`。

---

## 3. status：查看服务状态

查看 Nginx 状态：

```bash
systemctl status nginx
```

重点看：

| 字段 | 含义 |
| --- | --- |
| `Loaded` | 服务配置是否加载 |
| `Active` | 当前是否运行 |
| `Main PID` | 主进程 PID |
| `Docs` | 文档 |

只看是否运行：

```bash
systemctl is-active nginx
```

---

## 4. start / stop：启动和停止服务

启动：

```bash
sudo systemctl start nginx
```

查看：

```bash
systemctl status nginx
```

停止：

```bash
sudo systemctl stop nginx
```

再次查看：

```bash
systemctl status nginx
```

---

## 5. restart / reload：重启和重载

重启服务：

```bash
sudo systemctl restart nginx
```

重载配置：

```bash
sudo systemctl reload nginx
```

区别：

| 命令 | 含义 |
| --- | --- |
| `restart` | 停止后重新启动 |
| `reload` | 不停止服务，重新加载配置 |

不是所有服务都支持 `reload`。

---

## 6. enable / disable：开机自启

设置开机自启：

```bash
sudo systemctl enable nginx
```

查看是否自启：

```bash
systemctl is-enabled nginx
```

取消开机自启：

```bash
sudo systemctl disable nginx
```

再次查看：

```bash
systemctl is-enabled nginx
```

注意：

```text
enable 不等于 start。
disable 不等于 stop。
```

`enable` 管开机是否自动启动，`start` 管现在是否启动。

---

## 7. list-units：列出服务

列出正在运行的服务：

```bash
systemctl list-units --type=service --state=running
```

列出所有服务：

```bash
systemctl list-units --type=service --all
```

查找包含 nginx 的服务：

```bash
systemctl list-units --type=service --all | grep nginx
```

---

## 8. 本节小结

必须记住：

```bash
systemctl status nginx
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl enable nginx
sudo systemctl disable nginx
systemctl is-active nginx
systemctl is-enabled nginx
systemctl list-units --type=service --state=running
```

完成后进入下一节：`06-journalctl-logs.md`。

