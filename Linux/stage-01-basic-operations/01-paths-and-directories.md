# 01：路径、目录与文件系统

本节目标：学会确认当前位置、查看目录内容、切换目录，并理解 Linux 文件系统的基本结构。

本节命令顺序：

```text
1. pwd
2. ls
3. cd
4. tree
```

---

## 1. 理解 Linux 文件系统

Linux 的文件系统像一棵树，最顶层叫根目录：

```text
/
```

常见目录：

| 目录 | 作用 |
| --- | --- |
| `/` | 根目录，所有目录的起点 |
| `/home` | 普通用户的家目录 |
| `/etc` | 系统和软件配置文件 |
| `/var` | 日志、缓存、运行时数据 |
| `/usr` | 用户程序和共享资源 |
| `/bin` | 基础命令程序 |
| `/tmp` | 临时文件 |

初学阶段最常用的是自己的家目录：

```text
/home/你的用户名
```

它也可以简写为：

```text
~
```

---

## 2. 第一个命令：pwd

`pwd` 用来显示当前所在目录。

执行：

```bash
pwd
```

你可能看到：

```text
/home/study
```

这表示你当前位于 `study` 用户的家目录。

### 练习

执行：

```bash
cd ~
pwd
```

思考：

- `~` 表示什么？
- `pwd` 输出的是绝对路径还是相对路径？

答案：`~` 表示当前用户的家目录，`pwd` 输出的是绝对路径。

---

## 3. 第二个命令：ls

`ls` 用来查看目录内容。

最简单用法：

```bash
ls
```

查看更详细的信息：

```bash
ls -l
```

显示隐藏文件：

```bash
ls -a
```

更常用的组合：

```bash
ls -lah
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-l` | 以长列表格式显示 |
| `-a` | 显示隐藏文件 |
| `-h` | 文件大小用更容易读的单位显示 |

隐藏文件通常以点开头，例如：

```text
.bashrc
.profile
```

### 练习

依次执行：

```bash
cd ~
ls
ls -l
ls -a
ls -lah
```

观察每条命令输出有什么变化。

---

## 4. 第三个命令：cd

`cd` 用来切换目录。

回到家目录：

```bash
cd ~
```

进入根目录：

```bash
cd /
```

进入上一级目录：

```bash
cd ..
```

进入当前目录：

```bash
cd .
```

回到上一次所在目录：

```bash
cd -
```

### 重要路径符号

| 符号 | 含义 |
| --- | --- |
| `/` | 根目录 |
| `~` | 当前用户家目录 |
| `.` | 当前目录 |
| `..` | 上一级目录 |
| `-` | 上一次所在目录 |

### 练习

依次执行：

```bash
cd ~
pwd
cd /
pwd
cd /home
pwd
cd ..
pwd
cd -
pwd
```

每执行一次 `cd` 后，都用 `pwd` 确认自己到了哪里。

---

## 5. 绝对路径与相对路径

绝对路径：从根目录 `/` 开始写的路径。

示例：

```text
/home/study/linux-practice
```

相对路径：从当前目录出发写的路径。

示例：

```text
linux-practice/stage-01
```

如果你当前在 `/home/study`，那么：

```bash
cd linux-practice/stage-01
```

等价于：

```bash
cd /home/study/linux-practice/stage-01
```

### 练习

先回到家目录：

```bash
cd ~
```

用相对路径进入练习目录：

```bash
cd linux-practice/stage-01
pwd
```

再用绝对路径进入同一个目录：

```bash
cd ~/linux-practice/stage-01
pwd
```

---

## 6. 第四个命令：tree

`tree` 用树状结构显示目录。

如果没有安装，先执行：

```bash
sudo apt update
sudo apt install tree
```

在练习目录执行：

```bash
cd ~/linux-practice/stage-01
tree
```

显示当前目录和子目录，最多显示 2 层：

```bash
tree -L 2
```

### 练习

创建几个目录后再查看：

```bash
cd ~/linux-practice/stage-01
mkdir -p demo/a demo/b demo/c
tree
tree -L 2
```

---

## 7. 本节小结

本节必须记住：

```bash
pwd
ls -lah
cd ~
cd ..
cd -
tree -L 2
```

最重要的概念：

- `pwd`：我在哪里？
- `ls`：这里有什么？
- `cd`：我要去哪里？
- 绝对路径：从 `/` 开始。
- 相对路径：从当前位置开始。

完成后进入下一节：`02-create-copy-move-delete.md`。

