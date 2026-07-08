# 01：软件包管理

本节目标：学会在 Linux 中搜索、查看、安装、升级和卸载软件包。

本节命令顺序：

```text
1. apt update
2. apt search
3. apt show
4. apt install
5. dpkg -l
6. apt remove
7. apt autoremove
8. dnf / yum 对照
```

Ubuntu / Debian 系统主要使用 `apt`。Rocky Linux、CentOS、Fedora 常用 `dnf` 或 `yum`。

---

## 1. 软件包管理器是什么

软件包管理器负责：

- 从软件仓库下载软件。
- 自动处理依赖。
- 安装、升级、卸载软件。
- 查询软件包信息。

在 Ubuntu / Debian 中，常用组合是：

```text
apt：面向用户的包管理命令
dpkg：底层 Debian 包管理工具
```

---

## 2. apt update：更新软件源索引

安装软件前，先更新软件源索引：

```bash
sudo apt update
```

它不会升级所有软件，只是更新“仓库里有哪些软件、有哪些版本”的本地列表。

---

## 3. apt search：搜索软件

搜索软件包：

```bash
apt search tree
```

搜索 Nginx：

```bash
apt search nginx
```

如果结果太多，可以配合 `grep`：

```bash
apt search nginx | grep "^nginx"
```

---

## 4. apt show：查看软件包详情

查看软件包信息：

```bash
apt show tree
```

重点看：

- `Package`：包名
- `Version`：版本
- `Installed-Size`：安装后大小
- `Depends`：依赖
- `Description`：描述

---

## 5. apt install：安装软件

安装 `tree`：

```bash
sudo apt install tree
```

验证：

```bash
tree --version
```

安装 `htop`：

```bash
sudo apt install htop
```

验证：

```bash
htop --version
```

如果只是练习，推荐安装这些安全的小工具：

```bash
sudo apt install tree htop curl
```

---

## 6. dpkg -l：查看已安装软件包

查看所有已安装包：

```bash
dpkg -l
```

搜索某个包是否安装：

```bash
dpkg -l | grep tree
```

查看某个命令来自哪个包：

```bash
dpkg -S /usr/bin/tree
```

---

## 7. apt remove：卸载软件

卸载软件，但保留配置文件：

```bash
sudo apt remove tree
```

完全卸载，包括配置文件：

```bash
sudo apt purge tree
```

清理不再需要的依赖：

```bash
sudo apt autoremove
```

练习时可以卸载再装回：

```bash
sudo apt remove tree
sudo apt install tree
```

---

## 8. apt upgrade：升级软件

查看可升级软件：

```bash
apt list --upgradable
```

升级已安装软件：

```bash
sudo apt upgrade
```

在真实服务器上，升级前要了解影响范围，尤其是数据库、Web 服务、运行时环境。

---

## 9. dnf / yum 对照

如果你使用 Rocky Linux、CentOS 或 Fedora，可以参考：

| Ubuntu / Debian | Rocky / CentOS / Fedora |
| --- | --- |
| `sudo apt update` | `sudo dnf check-update` |
| `apt search nginx` | `dnf search nginx` |
| `apt show nginx` | `dnf info nginx` |
| `sudo apt install nginx` | `sudo dnf install nginx` |
| `sudo apt remove nginx` | `sudo dnf remove nginx` |
| `dpkg -l` | `rpm -qa` |

较新的系统多用 `dnf`，较老的 CentOS 可能用 `yum`。

---

## 10. 本节小结

必须记住：

```bash
sudo apt update
apt search package
apt show package
sudo apt install package
dpkg -l | grep package
dpkg -S /path/to/command
sudo apt remove package
sudo apt autoremove
apt list --upgradable
```

完成后进入下一节：`02-process-view.md`。

