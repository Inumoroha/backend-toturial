# Linux 系统学习路线

适合目标：从零开始系统掌握 Linux，最终能熟练使用命令行、管理系统、排查常见问题，并具备继续学习服务器运维、云原生、网络安全或嵌入式 Linux 的基础。

建议周期：12 周。  
建议节奏：每天 1 到 2 小时，每周至少完成 2 次动手实验。  
推荐环境：Windows + WSL2 Ubuntu，或虚拟机安装 Ubuntu Server / Debian / Rocky Linux。

---

## 0. 学习准备

### 目标

- 搭建一个安全、可反复练习的 Linux 环境。
- 熟悉终端、Shell、命令执行方式。
- 建立“边学边查、边做边记”的习惯。

### 推荐环境

任选其一：

1. WSL2 + Ubuntu：适合 Windows 用户，轻量方便。
2. VirtualBox / VMware + Ubuntu Server：更接近真实服务器。
3. 云服务器：适合后期学习 SSH、部署、网络和服务管理。

### 必备工具

- 一个 Linux 发行版：Ubuntu / Debian / Rocky Linux / Fedora 均可。
- 一个终端：Windows Terminal / VS Code Terminal / Linux 原生终端。
- 一个笔记文件：记录常用命令、错误原因、排查过程。

### 本阶段练习

```bash
whoami
pwd
ls
date
uname -a
cat /etc/os-release
```

### 验收标准

- 能打开 Linux 终端。
- 能确认当前用户、当前路径、系统版本。
- 知道命令、参数、路径、标准输出的大致含义。

---

## 第 1 阶段：Linux 基础操作

建议时间：第 1 到 2 周

### 核心目标

- 理解 Linux 文件系统结构。
- 熟练进行文件和目录操作。
- 掌握命令帮助系统。

### 必学知识

- 文件系统层级：`/`、`/home`、`/etc`、`/var`、`/usr`、`/bin`、`/tmp`
- 绝对路径与相对路径
- 常见文件类型
- 命令格式：`command [options] [arguments]`
- 帮助命令：`man`、`--help`、`info`

### 必会命令

```bash
pwd
ls
cd
mkdir
rmdir
touch
cp
mv
rm
cat
less
head
tail
file
stat
tree
man
```

### 动手练习

1. 在家目录下创建如下结构：

```text
linux-practice/
  notes/
  scripts/
  logs/
  backup/
```

2. 创建 5 个文本文件，并练习复制、移动、重命名和删除。
3. 使用 `man ls` 查看 `ls -l`、`ls -a`、`ls -h` 的含义。
4. 使用 `tail -f` 实时查看一个日志文件。

### 验收标准

- 能不依赖图形界面完成文件管理。
- 能解释常见目录的用途。
- 遇到陌生命令时，知道如何查帮助。

---

## 第 2 阶段：文本处理与管道

建议时间：第 3 到 4 周

### 核心目标

- 掌握 Linux 命令行最重要的组合能力：管道、重定向、文本过滤。
- 能快速查看、搜索、统计和处理文本。

### 必学知识

- 标准输入、标准输出、标准错误
- 输出重定向：`>`、`>>`
- 输入重定向：`<`
- 管道：`|`
- 通配符：`*`、`?`、`[]`
- 基础正则表达式

### 必会命令

```bash
grep
find
locate
wc
sort
uniq
cut
tr
sed
awk
xargs
tee
```

### 动手练习

1. 从 `/var/log` 中查找包含 `error` 的日志行。
2. 统计一个文件的行数、单词数、字符数。
3. 从一份 CSV 文本中提取第 1 列和第 3 列。
4. 使用 `find` 找出最近 7 天修改过的文件。
5. 使用管道组合命令，例如：

```bash
cat access.log | grep "404" | awk '{print $1}' | sort | uniq -c | sort -nr
```

### 验收标准

- 能使用管道把多个命令组合起来。
- 能用 `grep`、`find`、`awk` 解决常见文本查询问题。
- 能看懂简单的一行 Shell 数据处理命令。

---

## 第 3 阶段：用户、权限与文件安全

建议时间：第 5 周

### 核心目标

- 理解 Linux 的用户、用户组和权限模型。
- 能正确使用 `sudo`，避免误操作。
- 能排查“Permission denied”类问题。

### 必学知识

- 用户与用户组
- 文件权限：读、写、执行
- 权限表示法：`rwx`、数字权限 `755` / `644`
- 文件所有者与所属组
- `sudo` 与 root 用户
- 特殊权限：SUID、SGID、Sticky Bit

### 必会命令

```bash
id
who
groups
useradd
usermod
passwd
groupadd
chown
chgrp
chmod
sudo
su
umask
```

### 动手练习

1. 创建一个新用户 `devuser`。
2. 创建一个新用户组 `project`。
3. 设置一个目录只有 `project` 组成员可以写入。
4. 分别设置文件权限为 `644`、`600`、`755`，观察差异。
5. 故意制造一次权限错误，然后通过 `ls -l` 排查。

### 验收标准

- 能解释 `-rw-r--r--` 的含义。
- 能合理设置文件、脚本、目录权限。
- 知道什么时候需要 `sudo`，什么时候不应该使用。

---

## 第 4 阶段：软件包、进程与服务管理

建议时间：第 6 到 7 周

### 核心目标

- 能安装、升级、卸载软件。
- 能查看和管理进程。
- 能理解系统服务的启动、停止和开机自启。

### 必学知识

- 软件包管理器：`apt`、`dnf`、`yum`
- 进程、PID、前台、后台
- 信号：`SIGTERM`、`SIGKILL`
- systemd 与 service
- 日志查看基础

### 必会命令

```bash
apt update
apt install
apt remove
dpkg -l
ps
top
htop
pgrep
kill
killall
jobs
fg
bg
nohup
systemctl
journalctl
```

### 动手练习

1. 安装 `nginx` 或 `curl`。
2. 查看正在运行的进程，并找到某个进程的 PID。
3. 启动、停止、重启一个系统服务。
4. 查看某个服务的日志。

示例：

```bash
sudo apt update
sudo apt install nginx
systemctl status nginx
sudo systemctl restart nginx
journalctl -u nginx --since "1 hour ago"
```

### 验收标准

- 能安装常用软件。
- 能判断服务是否正在运行。
- 能通过进程和日志定位基础问题。

---

## 第 5 阶段：Shell 脚本入门

建议时间：第 8 到 9 周

### 核心目标

- 能编写简单 Shell 脚本自动化重复任务。
- 理解变量、条件、循环、函数和退出状态。

### 必学知识

- Shebang：`#!/usr/bin/env bash`
- 变量与引用
- 条件判断：`if`
- 循环：`for`、`while`
- 函数
- 脚本参数：`$1`、`$2`、`$@`、`$#`
- 退出码：`$?`
- 调试：`bash -x`

### 必会语法

```bash
#!/usr/bin/env bash

name="$1"

if [ -z "$name" ]; then
  echo "Usage: $0 <name>"
  exit 1
fi

echo "Hello, $name"
```

### 动手练习

1. 写一个脚本，自动创建项目目录结构。
2. 写一个脚本，备份指定目录到 `backup/`。
3. 写一个脚本，检查某个服务是否运行。
4. 写一个脚本，统计日志中出现最多的 IP。

### 验收标准

- 能独立写出 30 到 80 行的 Shell 脚本。
- 能给脚本添加执行权限并运行。
- 能处理参数缺失、文件不存在等常见异常。

---

## 第 6 阶段：网络基础与远程管理

建议时间：第 10 周

### 核心目标

- 理解 Linux 网络配置和常用排查方式。
- 能使用 SSH 连接远程服务器。
- 能进行基础端口、DNS、连通性排查。

### 必学知识

- IP、端口、协议、网关、DNS
- TCP 与 UDP 基础
- SSH 登录与密钥认证
- 防火墙基础
- 常见网络排查路径

### 必会命令

```bash
ip addr
ip route
ping
curl
wget
ss
netstat
dig
nslookup
traceroute
ssh
scp
rsync
ufw
```

### 动手练习

1. 查看本机 IP 地址和默认网关。
2. 使用 `curl` 请求一个网站并查看响应头。
3. 使用 `ss -tunlp` 查看监听端口。
4. 配置 SSH 密钥登录一台远程机器或本地虚拟机。
5. 使用 `scp` 或 `rsync` 复制文件。

### 验收标准

- 能判断网络不通大概是 DNS、路由、端口还是服务问题。
- 能通过 SSH 安全登录远程 Linux。
- 能查看本机开放了哪些端口。

---

## 第 7 阶段：磁盘、文件系统与系统资源

建议时间：第 11 周

### 核心目标

- 能查看磁盘、内存、CPU 使用情况。
- 能理解挂载、分区、文件系统的基本概念。
- 能排查磁盘空间不足问题。

### 必学知识

- 磁盘、分区、文件系统
- 挂载点
- inode
- swap
- CPU、内存、IO 基础指标

### 必会命令

```bash
df
du
lsblk
mount
umount
free
vmstat
iostat
lsof
ncdu
```

### 动手练习

1. 查看根分区剩余空间。
2. 找出当前目录下最大的 10 个文件或目录。
3. 查看内存使用情况。
4. 制造一个大文件并观察磁盘占用变化。

示例：

```bash
du -ah . | sort -rh | head -n 10
df -h
free -h
```

### 验收标准

- 能定位哪个目录占用了大量空间。
- 知道 `df` 和 `du` 的区别。
- 能解释内存、swap、磁盘空间的基础状态。

---

## 第 8 阶段：综合实战项目

建议时间：第 12 周

### 项目 1：个人 Linux 工具箱

创建一个目录 `linux-toolkit/`，里面包含：

```text
linux-toolkit/
  scripts/
    backup.sh
    check-service.sh
    top-ip.sh
  notes/
    commands.md
    troubleshooting.md
  logs/
  README.md
```

要求：

- `backup.sh` 可以备份指定目录。
- `check-service.sh` 可以检查服务是否运行。
- `top-ip.sh` 可以统计日志中访问最多的 IP。
- `commands.md` 记录你最常用的 50 个命令。
- `troubleshooting.md` 记录至少 5 个排错案例。

### 项目 2：部署一个简单 Web 服务

目标：

- 安装 Nginx。
- 修改默认首页。
- 启动服务并设置开机自启。
- 查看访问日志和错误日志。
- 使用 `curl` 验证服务。

验收命令：

```bash
systemctl status nginx
curl -I http://localhost
journalctl -u nginx --since "today"
```

### 项目 3：模拟线上问题排查

模拟问题：

- 服务没有启动。
- 端口被占用。
- 磁盘空间不足。
- 文件权限错误。
- DNS 解析失败。

每个问题记录：

- 现象是什么？
- 你用了哪些命令？
- 根因是什么？
- 最终如何修复？

---

## 建议学习顺序总览

| 周数 | 主题 | 重点 |
| --- | --- | --- |
| 第 1 周 | 环境与文件系统 | 终端、路径、目录结构 |
| 第 2 周 | 文件操作 | 增删改查、帮助系统 |
| 第 3 周 | 文本处理 | grep、find、管道 |
| 第 4 周 | 数据处理 | awk、sed、xargs |
| 第 5 周 | 用户与权限 | chmod、chown、sudo |
| 第 6 周 | 软件与进程 | apt、ps、top、kill |
| 第 7 周 | 服务管理 | systemctl、journalctl |
| 第 8 周 | Shell 脚本 | 变量、条件、循环 |
| 第 9 周 | 脚本实战 | 备份、日志统计、服务检查 |
| 第 10 周 | 网络与 SSH | ip、ss、curl、ssh |
| 第 11 周 | 磁盘与资源 | df、du、free、lsblk |
| 第 12 周 | 综合项目 | 部署、自动化、排错 |

---

## 每日学习模板

每天可以按这个顺序学习：

1. 学 3 到 5 个新命令。
2. 查看每个命令的帮助文档。
3. 自己构造 3 个使用场景。
4. 把命令和示例记录到笔记。
5. 做一次小实验。
6. 记录遇到的错误和解决过程。

示例笔记格式：

````markdown
## 命令：grep

用途：从文本中搜索匹配内容。

常用参数：

- `-i`：忽略大小写
- `-n`：显示行号
- `-r`：递归搜索

示例：

```bash
grep -in "error" app.log
```

踩坑：

- 搜索目录时需要加 `-r`。
- 搜索特殊字符时要注意引号。
````

---

## 常见命令速查

### 文件与目录

```bash
ls -lah
cd /path/to/dir
mkdir -p a/b/c
cp source target
mv old new
rm file
rm -r dir
```

### 查看文本

```bash
cat file
less file
head file
tail file
tail -f log.txt
```

### 搜索与统计

```bash
grep "keyword" file
find . -name "*.log"
wc -l file
sort file
uniq -c
```

### 权限

```bash
ls -l
chmod 755 script.sh
chown user:group file
sudo command
```

### 进程与服务

```bash
ps aux
top
kill PID
systemctl status nginx
journalctl -u nginx
```

### 网络

```bash
ip addr
ping example.com
curl -I https://example.com
ss -tunlp
ssh user@host
```

### 磁盘与资源

```bash
df -h
du -sh *
free -h
lsblk
```

---

## 推荐练习原则

- 不要只背命令，要用命令解决具体问题。
- 每个命令至少亲手运行 3 次。
- 每次报错都记录下来，错误是学习 Linux 的捷径。
- 重要命令先在测试目录里练习，尤其是 `rm`、`chmod`、`chown`。
- 少复制整段命令，多拆开理解每一步。
- 学会阅读日志，比记住更多命令更重要。

---

## 后续进阶方向

掌握本路线后，可以按兴趣选择一个方向继续深入：

### 运维 / SRE

- systemd 深入
- 日志管理
- 监控告警
- Nginx
- Docker
- Kubernetes
- CI/CD

### 后端开发

- Linux 开发环境
- Git
- Makefile
- 进程与端口排查
- 服务部署
- 性能分析

### 网络安全

- Linux 权限模型
- 日志分析
- 网络扫描
- 防火墙
- SSH 安全
- 基础加固

### 嵌入式 Linux

- 交叉编译
- BusyBox
- 内核模块
- 设备树
- 文件系统裁剪

---

## 最小毕业标准

当你能独立完成下面任务时，就说明 Linux 基础已经比较扎实：

1. 在命令行中完成文件管理。
2. 使用 `grep`、`find`、`awk` 分析文本。
3. 正确设置用户、用户组和文件权限。
4. 安装并管理一个系统服务。
5. 编写简单 Shell 脚本自动化任务。
6. 使用 SSH 管理远程服务器。
7. 排查服务、端口、权限、日志、磁盘空间等常见问题。
8. 能把自己的排查过程写成清晰的笔记。

---

## 建议的第一周任务清单

- [ ] 安装 WSL2 Ubuntu 或虚拟机 Linux。
- [ ] 熟悉 `pwd`、`ls`、`cd`。
- [ ] 创建 `linux-practice/` 练习目录。
- [ ] 学会使用 `man` 查看帮助。
- [ ] 完成 20 次文件创建、复制、移动、删除练习。
- [ ] 写一篇笔记：Linux 文件系统结构。
- [ ] 写一篇笔记：我最常用的 10 个命令。

---

## 结语

Linux 学习最重要的不是一次性记住所有命令，而是建立一套稳定的解决问题流程：

```text
观察现象 -> 查看状态 -> 阅读日志 -> 缩小范围 -> 验证假设 -> 修复问题 -> 记录复盘
```

只要坚持动手练习，Linux 会从“满屏陌生命令”逐渐变成一个非常可靠、清晰、强大的工作环境。
