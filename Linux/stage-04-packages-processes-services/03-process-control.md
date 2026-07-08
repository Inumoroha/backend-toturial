# 03：控制进程

本节目标：理解信号，学会用 `kill` 和 `killall` 正常结束进程。

本节命令顺序：

```text
1. pgrep -af
2. kill PID
3. kill -TERM PID
4. kill -KILL PID
5. kill -l
6. killall name
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-04
```

---

## 1. 信号是什么

Linux 通过信号和进程通信。结束进程本质上是给进程发送信号。

常见信号：

| 信号 | 编号 | 含义 |
| --- | --- | --- |
| `SIGTERM` | `15` | 请求进程正常退出 |
| `SIGKILL` | `9` | 强制杀死进程 |
| `SIGHUP` | `1` | 挂起或重新加载配置 |
| `SIGINT` | `2` | 中断，类似 `Ctrl + C` |

优先使用 `SIGTERM`，不要一上来就 `kill -9`。

---

## 2. 启动一个练习进程

执行：

```bash
./scripts/loop.sh > logs/loop.log 2>&1 &
```

查看 PID：

```bash
pgrep -af loop.sh
```

查看日志：

```bash
tail -n 5 logs/loop.log
```

---

## 3. kill：正常结束进程

先找到 PID：

```bash
pgrep -af loop.sh
```

假设 PID 是 `12345`，执行：

```bash
kill 12345
```

`kill PID` 默认发送 `SIGTERM`。

确认是否结束：

```bash
pgrep -af loop.sh
```

如果没有输出，说明进程已经结束。

---

## 4. kill -TERM：明确发送正常退出信号

重新启动练习进程：

```bash
./scripts/loop.sh > logs/loop.log 2>&1 &
pgrep -af loop.sh
```

发送 `SIGTERM`：

```bash
kill -TERM PID
```

请把 `PID` 替换成真实数字。

也可以使用编号：

```bash
kill -15 PID
```

---

## 5. kill -KILL：强制杀死

如果进程无法正常退出，最后再使用：

```bash
kill -KILL PID
```

等价于：

```bash
kill -9 PID
```

注意：

```text
SIGKILL 不能被进程捕获，进程没有机会清理临时文件或保存状态。
```

所以真实工作中不要滥用 `kill -9`。

---

## 6. kill -l：查看信号列表

执行：

```bash
kill -l
```

你会看到系统支持的信号列表。

常记这几个就够了：

```text
1 HUP
2 INT
9 KILL
15 TERM
```

---

## 7. killall：按名称结束进程

`killall` 按进程名发送信号。

启动两个练习进程：

```bash
./scripts/loop.sh > logs/loop-1.log 2>&1 &
./scripts/loop.sh > logs/loop-2.log 2>&1 &
pgrep -af loop.sh
```

结束所有匹配的 `loop.sh`：

```bash
killall loop.sh
```

如果你的系统没有 `killall`，可以安装：

```bash
sudo apt install psmisc
```

谨慎使用 `killall`，因为它会影响所有同名进程。

---

## 8. 本节小结

必须记住：

```bash
pgrep -af name
kill PID
kill -TERM PID
kill -15 PID
kill -KILL PID
kill -9 PID
kill -l
killall name
```

推荐顺序：

```text
先查 PID -> kill PID -> 等待 -> 还不退出再考虑 kill -9。
```

完成后进入下一节：`04-foreground-background.md`。

