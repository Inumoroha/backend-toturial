# Linux 学习准备阶段教程

本教程对应 Linux 系统学习路线中的“0. 学习准备”阶段。目标不是一次性学会很多命令，而是先搭建一个稳定、安全、方便反复练习的 Linux 环境，并理解终端的基本使用方式。

适合人群：

- 从零开始学习 Linux 的新手。
- 使用 Windows 电脑，想通过 WSL2 学习 Linux。
- 想先搭建虚拟机，再逐步接触服务器环境的人。

预计用时：1 到 2 天。  
阶段目标：完成 Linux 环境搭建，并能运行基础命令确认系统状态。

---

## 1. 准备阶段要完成什么

准备阶段需要完成 4 件事：

1. 选择一种 Linux 练习环境。
2. 安装并进入 Linux 终端。
3. 理解最基本的命令行概念。
4. 完成第一次命令练习。

完成后，你应该能做到：

- 打开 Linux 终端。
- 知道自己当前在哪个目录。
- 知道当前使用的是哪个用户。
- 查看 Linux 系统版本。
- 创建一个后续学习用的练习目录。

---

## 2. 选择 Linux 学习环境

常见选择有三种：WSL2、虚拟机、云服务器。

### 方案一：WSL2 + Ubuntu

推荐指数：高。  
适合人群：Windows 用户、初学者、想快速开始的人。

优点：

- 安装方便。
- 启动速度快。
- 和 Windows 文件可以互相访问。
- 不容易把真实系统弄坏。

缺点：

- 和真实服务器环境有少量差异。
- systemd、网络、防火墙等内容在某些版本中表现可能不同。

如果你主要是想学习命令行、Shell、文件系统、文本处理，WSL2 非常合适。

### 方案二：虚拟机 + Linux

推荐指数：高。  
适合人群：想更接近真实服务器环境的人。

常用软件：

- VirtualBox
- VMware Workstation Player
- Hyper-V

推荐发行版：

- Ubuntu Server
- Debian
- Rocky Linux

优点：

- 更接近真实 Linux 主机。
- 适合学习网络、磁盘、服务管理。
- 可以创建快照，出错后恢复。

缺点：

- 占用内存和磁盘更多。
- 安装步骤比 WSL2 稍多。

### 方案三：云服务器

推荐指数：中。  
适合人群：已经有一点基础，想学习远程管理和服务部署的人。

优点：

- 最接近真实线上环境。
- 适合学习 SSH、Nginx、防火墙、部署。

缺点：

- 通常需要付费。
- 新手容易因为误操作导致安全问题。
- 暴露在公网，需要重视密码、端口和密钥管理。

### 初学者建议

如果你使用 Windows，建议先用：

```text
WSL2 + Ubuntu
```

学到网络、服务、磁盘阶段后，再补一个：

```text
虚拟机 Ubuntu Server
```

这样开始轻，后期又能接近真实服务器。

---

## 3. 在 Windows 上安装 WSL2 Ubuntu

如果你已经有 Linux 环境，可以跳过本节。

### 3.1 检查 Windows 版本

WSL2 需要较新的 Windows 10 或 Windows 11。

在 PowerShell 中执行：

```powershell
winver
```

如果是 Windows 11，通常可以直接安装。  
如果是 Windows 10，建议先更新到较新的系统版本。

### 3.2 安装 WSL

用管理员身份打开 PowerShell，执行：

```powershell
wsl --install
```

执行后根据提示重启电脑。

重启后，系统通常会自动安装 Ubuntu。如果没有自动安装，可以执行：

```powershell
wsl --list --online
wsl --install -d Ubuntu
```

### 3.3 第一次启动 Ubuntu

安装完成后，在开始菜单搜索：

```text
Ubuntu
```

第一次打开时，它会要求创建 Linux 用户名和密码。

注意：

- Linux 用户名建议使用小写英文，例如 `study`、`dev`、`linuxuser`。
- 输入密码时，终端通常不会显示星号，这是正常现象。
- 这个密码是 Linux 内部用户密码，不一定等于 Windows 密码。

### 3.4 检查 WSL 版本

在 PowerShell 中执行：

```powershell
wsl -l -v
```

你应该能看到类似结果：

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

如果 `VERSION` 是 `2`，说明正在使用 WSL2。

如果是 `1`，可以执行：

```powershell
wsl --set-version Ubuntu 2
```

---

## 4. 虚拟机安装建议

如果你选择虚拟机，建议使用 Ubuntu Server。

### 4.1 推荐配置

最低配置：

- CPU：2 核
- 内存：2 GB
- 磁盘：20 GB

更舒服的配置：

- CPU：2 到 4 核
- 内存：4 GB
- 磁盘：40 GB

### 4.2 安装建议

安装时建议：

- 语言选择 English，方便查资料和阅读错误信息。
- 键盘选择默认布局即可。
- 用户名使用小写英文。
- 勾选安装 OpenSSH Server，后续方便用 SSH 登录。
- 不要一开始就分复杂的磁盘分区，先使用默认方案。

### 4.3 快照习惯

虚拟机最大的优势是可以创建快照。建议在这些时间点创建快照：

- 系统刚安装完成。
- 完成 SSH 配置后。
- 学习权限、磁盘、服务管理前。
- 做高风险实验前。

这样即使误删文件、改坏配置，也可以恢复。

---

## 5. 认识终端

Linux 主要通过终端使用。终端不是“黑窗口”，它是你和系统交互的入口。

你输入命令，系统执行命令，然后把结果输出给你。

一个典型命令长这样：

```bash
ls -lah /home
```

它可以拆成三部分：

```text
ls       命令名称
-lah     选项
/home    参数
```

含义是：以详细、包含隐藏文件、人类可读大小的方式查看 `/home` 目录。

### 命令提示符

你可能会看到类似提示符：

```text
study@ubuntu:~$
```

它通常表示：

```text
study     当前用户
ubuntu    主机名
~         当前目录
$         普通用户
```

如果最后是 `#`，通常表示当前是 root 用户：

```text
root@ubuntu:~#
```

初学阶段尽量使用普通用户，只在必要时使用 `sudo`。

---

## 6. 第一次命令练习

打开 Linux 终端，依次执行下面命令。

### 6.1 查看当前用户

```bash
whoami
```

作用：显示当前登录用户。

示例输出：

```text
study
```

### 6.2 查看当前目录

```bash
pwd
```

作用：显示你当前所在的位置。

示例输出：

```text
/home/study
```

### 6.3 查看目录内容

```bash
ls
```

显示当前目录下的文件和目录。

更常用的写法：

```bash
ls -lah
```

参数含义：

- `-l`：使用详细列表格式。
- `-a`：显示隐藏文件。
- `-h`：用更容易阅读的单位显示大小。

### 6.4 查看日期时间

```bash
date
```

作用：显示系统当前时间。

### 6.5 查看内核信息

```bash
uname -a
```

作用：显示内核、架构等系统信息。

你不需要立刻理解所有输出，先知道它是用来查看系统底层信息的。

### 6.6 查看发行版信息

```bash
cat /etc/os-release
```

作用：查看当前 Linux 发行版。

你可能会看到：

```text
NAME="Ubuntu"
VERSION="24.04 LTS (Noble Numbat)"
ID=ubuntu
```

不同版本显示可能不同，这是正常的。

---

## 7. 创建学习目录

后续学习建议统一放在一个练习目录里，避免到处创建文件。

执行：

```bash
mkdir -p ~/linux-practice/{notes,scripts,logs,backup}
```

这条命令会创建：

```text
linux-practice/
  notes/
  scripts/
  logs/
  backup/
```

进入目录：

```bash
cd ~/linux-practice
```

查看结构：

```bash
ls -lah
```

如果安装了 `tree`，也可以执行：

```bash
tree ~/linux-practice
```

如果没有 `tree`，可以安装：

```bash
sudo apt update
sudo apt install tree
```

---

## 8. 学会使用帮助

Linux 不要求你记住所有命令。更重要的是知道如何查。

### 8.1 使用 `--help`

```bash
ls --help
```

适合快速查看命令参数。

### 8.2 使用 `man`

```bash
man ls
```

`man` 是 manual 的缩写，表示手册。

进入 `man` 页面后：

- 按 `j` 或方向键向下移动。
- 按 `k` 或方向键向上移动。
- 按 `/关键词` 搜索。
- 按 `n` 跳到下一个搜索结果。
- 按 `q` 退出。

### 8.3 使用 `type`

```bash
type cd
type ls
```

作用：查看一个命令是 Shell 内置命令，还是外部程序。

---

## 9. 终端常用快捷键

这些快捷键能明显提升效率。

| 快捷键 | 作用 |
| --- | --- |
| `Ctrl + C` | 中断当前命令 |
| `Ctrl + L` | 清屏 |
| `Ctrl + A` | 光标移动到行首 |
| `Ctrl + E` | 光标移动到行尾 |
| `Ctrl + U` | 删除光标前的内容 |
| `Ctrl + K` | 删除光标后的内容 |
| `Tab` | 自动补全命令或路径 |
| `上下方向键` | 查看历史命令 |

最常用的是：

```text
Tab、Ctrl + C、Ctrl + L、上下方向键
```

---

## 10. sudo 是什么

有些命令需要管理员权限，例如安装软件：

```bash
sudo apt install tree
```

`sudo` 的意思是以管理员权限执行这条命令。

第一次使用 `sudo` 时，会要求输入当前用户密码。

注意：

- 输入密码时不会显示字符，这是正常现象。
- 不要随便给所有命令加 `sudo`。
- 特别小心 `sudo rm`、`sudo chmod`、`sudo chown`。

初学阶段可以记住一句话：

```text
看不懂的命令，不要随便加 sudo 执行。
```

---

## 11. 常见问题

### 问题 1：输入密码时没有任何显示

这是正常现象。Linux 终端输入密码时，通常不会显示星号或圆点。

直接输入密码，然后按 Enter 即可。

### 问题 2：提示 command not found

示例：

```text
tree: command not found
```

原因通常是命令没有安装。

Ubuntu / Debian 可以尝试：

```bash
sudo apt update
sudo apt install tree
```

### 问题 3：提示 Permission denied

示例：

```text
Permission denied
```

常见原因：

- 当前用户没有权限。
- 文件没有执行权限。
- 目录不允许进入或写入。

先查看权限：

```bash
ls -l
```

不要第一时间乱用 `sudo`，先理解是谁没有什么权限。

### 问题 4：不知道自己在哪个目录

执行：

```bash
pwd
```

回到家目录：

```bash
cd ~
```

### 问题 5：命令执行后没有输出

有些命令成功执行时不会输出任何内容。例如：

```bash
mkdir test
```

如果没有报错，通常就是成功了。

可以用 `ls` 验证：

```bash
ls
```

---

## 12. 准备阶段练习任务

请按顺序完成下面任务。

### 任务 1：确认系统信息

执行：

```bash
whoami
pwd
date
uname -a
cat /etc/os-release
```

把输出记录到笔记中。

### 任务 2：创建练习目录

执行：

```bash
mkdir -p ~/linux-practice/{notes,scripts,logs,backup}
cd ~/linux-practice
ls -lah
```

### 任务 3：创建第一篇笔记

执行：

```bash
cd ~/linux-practice/notes
touch day-01.md
```

编辑这个文件，记录：

- 今天使用的 Linux 环境。
- 当前系统版本。
- 学会的 5 个命令。
- 遇到的 1 个问题。

如果你还不会用 Linux 编辑器，可以先在 Windows 中编辑，后面再学习 `nano`、`vim` 或 VS Code。

### 任务 4：练习帮助命令

执行：

```bash
ls --help
man ls
```

找到下面参数的含义：

- `-l`
- `-a`
- `-h`
- `-R`

### 任务 5：练习终端快捷键

至少练习这些操作：

- 用方向键找回上一条命令。
- 用 `Tab` 自动补全路径。
- 用 `Ctrl + L` 清屏。
- 用 `Ctrl + C` 中断一条正在运行的命令。

可以用下面命令测试中断：

```bash
ping 8.8.8.8
```

运行后按 `Ctrl + C` 停止。

---

## 13. 准备阶段验收清单

完成下面清单后，就可以进入第 1 阶段“Linux 基础操作”。

- [x] 已安装 WSL2、虚拟机或其他 Linux 环境。
- [x] 能打开 Linux 终端。
- [x] 能执行 `whoami` 查看当前用户。
- [x] 能执行 `pwd` 查看当前目录。
- [x] 能执行 `ls -lah` 查看目录内容。
- [x] 能执行 `cat /etc/os-release` 查看系统版本。
- [x] 已创建 `~/linux-practice/` 练习目录。
- [x] 知道普通用户提示符 `$` 和 root 提示符 `#` 的区别。
- [x] 知道 `sudo` 是管理员权限，不会随便乱用。
- [x] 会使用 `--help` 或 `man` 查询命令帮助。

---

## 14. 下一步学习建议

准备阶段完成后，继续学习：

1. Linux 文件系统结构。
2. 文件和目录操作。
3. 命令帮助系统。
4. 路径、隐藏文件、文件类型。

建议你从这几个命令开始：

```bash
pwd
ls
cd
mkdir
touch
cp
mv
rm
cat
less
man
```

学习方式仍然是：

```text
先运行 -> 看输出 -> 查帮助 -> 改参数 -> 做笔记
```

只要准备阶段环境搭稳，后面学习 Linux 会顺很多。
