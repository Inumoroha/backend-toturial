# 06：lsof 查看打开文件和占用

本节目标：学会用 `lsof` 查看进程打开的文件，排查文件、目录、端口和已删除文件占用问题。

本节命令顺序：

```text
1. lsof
2. lsof file
3. lsof +D dir
4. lsof -p PID
5. lsof -i :PORT
6. lsof | grep deleted
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-07
```

---

## 1. lsof 是什么

`lsof` 表示 list open files。

在 Linux 中，很多东西都可以看作文件：

- 普通文件
- 目录
- 设备
- 网络 socket
- 管道

所以 `lsof` 能用于排查“谁正在使用这个文件或端口”。

---

## 2. 安装 lsof

如果没有安装：

```bash
sudo apt update
sudo apt install lsof
```

查看帮助：

```bash
lsof -v
```

---

## 3. 查看某个文件被谁打开

创建文件：

```bash
echo "hello lsof" > tmp/lsof-demo.log
```

用 `tail -f` 打开它：

```bash
tail -f tmp/lsof-demo.log
```

另开一个终端，进入练习目录：

```bash
cd ~/linux-practice/stage-07
```

查看谁打开了文件：

```bash
lsof tmp/lsof-demo.log
```

回到第一个终端，按：

```text
Ctrl + C
```

停止 `tail -f`。

---

## 4. 查看目录下被打开的文件

查看目录内打开文件：

```bash
lsof +D tmp
```

注意：

```text
lsof +D 会递归扫描目录，目录很大时可能比较慢。
```

---

## 5. 查看某个进程打开了什么

先启动一个进程：

```bash
tail -f tmp/lsof-demo.log &
```

查看 PID：

```bash
pgrep -af "tail -f tmp/lsof-demo.log"
```

假设 PID 是 `12345`，执行：

```bash
lsof -p 12345
```

结束进程：

```bash
kill 12345
```

---

## 6. 查看端口占用

查看 80 端口：

```bash
sudo lsof -i :80
```

查看 8080 端口：

```bash
sudo lsof -i :8080
```

启动临时 HTTP 服务：

```bash
python3 -m http.server 8080
```

另开终端：

```bash
sudo lsof -i :8080
```

回到服务终端按 `Ctrl + C` 停止。

---

## 7. 已删除但仍占用空间的文件

有时你删除了大日志，但 `df -h` 显示空间没有释放。可能是进程还打开着这个已删除文件。

排查：

```bash
sudo lsof | grep deleted
```

如果看到某个进程仍打开大文件，需要重启对应进程或让它重新打开日志。

注意：

```text
不要看到 deleted 就随便 kill 进程，先确认进程是什么服务。
```

---

## 8. 本节小结

必须记住：

```bash
lsof file
lsof +D dir
lsof -p PID
sudo lsof -i :80
sudo lsof -i :8080
sudo lsof | grep deleted
```

核心思路：

```text
文件删了但空间没释放，查 lsof deleted。
端口被占用，查 lsof -i :端口。
```

完成后进入下一节：`07-ncdu-cleanup.md`。

