# 04：前台、后台与持久运行

本节目标：理解前台任务、后台任务、作业控制，以及如何让命令在退出终端后继续运行。

本节命令顺序：

```text
1. command &
2. jobs
3. Ctrl + Z
4. bg
5. fg
6. nohup
7. tail -f
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-04
```

---

## 1. 前台任务

直接运行命令时，它占用当前终端，叫前台任务。

执行：

```bash
sleep 30
```

此时终端会等待 30 秒。

按下面快捷键可以中断：

```text
Ctrl + C
```

---

## 2. command &：后台运行

在命令末尾加 `&`，可以让它在后台运行：

```bash
sleep 60 &
```

查看当前 Shell 的后台任务：

```bash
jobs
```

输出类似：

```text
[1]+  Running                 sleep 60 &
```

---

## 3. Ctrl + Z：暂停前台任务

运行：

```bash
sleep 300
```

按：

```text
Ctrl + Z
```

任务会被暂停。

查看：

```bash
jobs
```

你可能看到：

```text
[1]+  Stopped                 sleep 300
```

---

## 4. bg：让暂停任务在后台继续

把暂停任务放到后台继续运行：

```bash
bg
```

查看：

```bash
jobs
```

---

## 5. fg：把后台任务调回前台

把最近的后台任务调回前台：

```bash
fg
```

如果有多个任务，可以指定编号：

```bash
fg %1
```

结束前台任务：

```text
Ctrl + C
```

---

## 6. nohup：退出终端后继续运行

`nohup` 可以让命令忽略挂起信号。

运行练习脚本：

```bash
nohup ./scripts/loop.sh > logs/nohup-loop.log 2>&1 &
```

查看进程：

```bash
pgrep -af loop.sh
```

查看日志：

```bash
tail -f logs/nohup-loop.log
```

停止 `tail -f`：

```text
Ctrl + C
```

结束练习进程：

```bash
pkill -f loop.sh
```

`pkill -f` 会按完整命令行匹配，使用前一定确认：

```bash
pgrep -af loop.sh
```

---

## 7. jobs 和 ps 的区别

| 命令 | 查看范围 |
| --- | --- |
| `jobs` | 当前 Shell 启动的后台/暂停任务 |
| `ps` | 系统进程 |
| `pgrep` | 按名称查系统进程 |

如果你换了一个终端，原终端的 `jobs` 不一定能看到，但 `ps` 和 `pgrep` 仍然可以看到进程。

---

## 8. 本节小结

必须记住：

```bash
command &
jobs
Ctrl + Z
bg
fg
fg %1
nohup command > file.log 2>&1 &
tail -f file.log
pgrep -af name
```

完成后进入下一节：`05-systemd-services.md`。

