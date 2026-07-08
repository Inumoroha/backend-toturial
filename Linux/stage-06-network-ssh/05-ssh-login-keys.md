# 05：SSH 登录与密钥

本节目标：学会使用 SSH 登录远程 Linux，理解用户名、主机、端口、密码登录和密钥登录。

本节命令顺序：

```text
1. ssh user@host
2. ssh -p port user@host
3. ssh-keygen
4. ssh-copy-id
5. ssh -i key user@host
6. ~/.ssh/config
```

本节最好在虚拟机、云服务器，或另一台 Linux 主机上练习。

---

## 1. SSH 是什么

SSH 用来安全地远程登录 Linux 主机。

基本格式：

```bash
ssh user@host
```

示例：

```bash
ssh ubuntu@192.168.1.100
```

含义：

| 部分 | 含义 |
| --- | --- |
| `ubuntu` | 远程用户名 |
| `192.168.1.100` | 远程主机 IP |
| `ssh` | 使用 SSH 协议连接 |

---

## 2. 连接指定端口

SSH 默认端口是 `22`。

如果远程 SSH 使用其他端口，例如 `2222`：

```bash
ssh -p 2222 user@host
```

第一次连接时，可能会出现主机指纹确认：

```text
Are you sure you want to continue connecting?
```

确认 IP 或域名正确后，输入：

```text
yes
```

---

## 3. 在练习机上安装 SSH 服务端

如果你要把当前 Linux 当成被连接的服务器：

```bash
sudo apt update
sudo apt install openssh-server
```

查看服务：

```bash
systemctl status ssh
```

启动服务：

```bash
sudo systemctl start ssh
```

查看监听端口：

```bash
sudo ss -tunlp | grep ssh
```

查看本机 IP：

```bash
hostname -I
```

然后从另一台机器连接：

```bash
ssh 你的用户名@服务器IP
```

WSL 中 SSH 服务端配置可能和虚拟机不同，新手更推荐用虚拟机练习。

---

## 4. ssh-keygen：创建密钥

SSH 密钥由两部分组成：

| 文件 | 含义 |
| --- | --- |
| 私钥 | 留在自己电脑上，不能泄露 |
| 公钥 | 放到远程服务器上 |

创建密钥：

```bash
ssh-keygen -t ed25519 -C "linux-study"
```

一路按 Enter 可以使用默认路径：

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

查看：

```bash
ls -l ~/.ssh
```

注意：

```text
不要把私钥 id_ed25519 发给别人。
公钥 id_ed25519.pub 可以放到服务器。
```

---

## 5. ssh-copy-id：复制公钥到服务器

把公钥复制到远程服务器：

```bash
ssh-copy-id user@host
```

如果 SSH 端口不是 22：

```bash
ssh-copy-id -p 2222 user@host
```

复制成功后，再连接：

```bash
ssh user@host
```

如果配置正确，就可以不输入密码，或只输入私钥 passphrase。

---

## 6. ssh -i：指定私钥

如果你的私钥不是默认文件名：

```bash
ssh -i ~/.ssh/my_key user@host
```

指定端口和私钥：

```bash
ssh -i ~/.ssh/my_key -p 2222 user@host
```

---

## 7. ~/.ssh/config：简化连接

创建或编辑：

```bash
nano ~/.ssh/config
```

写入示例：

```text
Host myserver
  HostName 192.168.1.100
  User ubuntu
  Port 22
  IdentityFile ~/.ssh/id_ed25519
```

设置权限：

```bash
chmod 600 ~/.ssh/config
```

以后连接：

```bash
ssh myserver
```

---

## 8. 常见 SSH 问题

### Permission denied

可能原因：

- 用户名不对。
- 密码不对。
- 公钥没有放到服务器。
- 服务器禁用了密码登录。
- 私钥权限不正确。

检查私钥权限：

```bash
chmod 600 ~/.ssh/id_ed25519
```

### Connection refused

可能原因：

- SSH 服务没启动。
- 端口不对。
- 防火墙拒绝。
- 目标主机没有监听 22 端口。

检查：

```bash
systemctl status ssh
sudo ss -tunlp | grep :22
```

### Connection timed out

可能原因：

- 网络不通。
- 防火墙丢弃连接。
- 云服务器安全组未开放端口。

---

## 9. 本节小结

必须记住：

```bash
ssh user@host
ssh -p 2222 user@host
ssh-keygen -t ed25519 -C "linux-study"
ssh-copy-id user@host
ssh -i ~/.ssh/id_ed25519 user@host
chmod 600 ~/.ssh/id_ed25519
nano ~/.ssh/config
ssh myserver
```

完成后进入下一节：`06-scp-rsync.md`。

