# 02：查看进程

本节目标：理解进程和 PID，学会使用 `ps`、`top`、`htop`、`pgrep` 查看正在运行的程序。

本节命令顺序：

```text
1. ps
2. ps aux
3. ps -ef
4. top
5. htop
6. pgrep
7. pidof
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-04
```

---

## 1. 进程是什么

程序是磁盘上的文件，进程是正在运行的程序实例。

例如：

```text
/usr/bin/bash 是程序
你打开的一个 bash 会话是进程
```

每个进程都有一个 PID：

```text
PID = Process ID
```

---

## 2. ps：查看当前终端相关进程

执行：

```bash
ps
```

常见列：

| 列 | 含义 |
| --- | --- |
| `PID` | 进程 ID |
| `TTY` | 终端 |
| `TIME` | 占用 CPU 时间 |
| `CMD` | 命令 |

---

## 3. ps aux：查看所有进程

执行：

```bash
ps aux
```

常见列：

| 列 | 含义 |
| --- | --- |
| `USER` | 运行进程的用户 |
| `PID` | 进程 ID |
| `%CPU` | CPU 占用 |
| `%MEM` | 内存占用 |
| `STAT` | 进程状态 |
| `COMMAND` | 启动命令 |

配合 `grep` 查找：

```bash
ps aux | grep bash
```

排除 grep 自身：

```bash
ps aux | grep bash | grep -v grep
```

---

## 4. ps -ef：另一种常见格式

执行：

```bash
ps -ef
```

查看父子进程关系时常用：

```bash
ps -ef | grep bash
```

常见列：

| 列 | 含义 |
| --- | --- |
| `UID` | 用户 |
| `PID` | 进程 ID |
| `PPID` | 父进程 ID |
| `CMD` | 命令 |

---

## 5. top：实时查看进程

执行：

```bash
top
```

常用按键：

| 按键 | 作用 |
| --- | --- |
| `q` | 退出 |
| `P` | 按 CPU 排序 |
| `M` | 按内存排序 |
| `k` | 杀进程 |
| `h` | 帮助 |

`top` 顶部信息包含系统负载、任务数量、CPU、内存等。

---

## 6. htop：更友好的进程查看工具

如果没有安装：

```bash
sudo apt install htop
```

执行：

```bash
htop
```

常用操作：

| 按键 | 作用 |
| --- | --- |
| `F3` | 搜索 |
| `F4` | 过滤 |
| `F6` | 排序 |
| `F9` | 发送信号 |
| `F10` | 退出 |

---

## 7. pgrep：按名称查 PID

查找 bash 进程：

```bash
pgrep bash
```

显示进程名：

```bash
pgrep -l bash
```

按完整命令行匹配：

```bash
pgrep -af bash
```

查找练习脚本：

```bash
./scripts/loop.sh > logs/loop.log 2>&1 &
pgrep -af loop.sh
```

---

## 8. pidof：查找程序 PID

查看某个程序的 PID：

```bash
pidof bash
```

如果程序没有运行，可能没有输出。

---

## 9. 本节小结

必须记住：

```bash
ps
ps aux
ps -ef
ps aux | grep name
top
htop
pgrep -l name
pgrep -af name
pidof name
```

核心概念：

```text
查进程时先找到 PID，再决定是否需要进一步操作。
```

完成后进入下一节：`03-process-control.md`。

