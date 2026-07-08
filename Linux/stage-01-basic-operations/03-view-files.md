# 03：查看文件内容与文件信息

本节目标：学会查看文本内容、文件开头、文件结尾、文件类型和文件详细信息。

本节命令顺序：

```text
1. cat
2. less
3. head
4. tail
5. file
6. stat
```

请先准备练习文件：

```bash
cd ~/linux-practice/stage-01
mkdir -p view-demo
cd view-demo
```

创建一个短文件：

```bash
printf "Linux\nShell\nFile\nDirectory\nCommand\n" > words.txt
```

创建一个稍长的文件：

```bash
seq 1 100 > numbers.txt
```

---

## 1. cat：直接显示文件内容

查看短文件：

```bash
cat words.txt
```

显示行号：

```bash
cat -n words.txt
```

`cat` 适合查看短文件，不适合查看特别长的文件。

### 练习

执行：

```bash
cat words.txt
cat -n words.txt
```

---

## 2. less：分页查看长文件

查看长文件：

```bash
less numbers.txt
```

进入 `less` 后常用按键：

| 按键 | 作用 |
| --- | --- |
| `空格` | 向下翻一页 |
| `b` | 向上翻一页 |
| `j` | 向下一行 |
| `k` | 向上一行 |
| `/关键词` | 搜索 |
| `n` | 下一个搜索结果 |
| `q` | 退出 |

### 练习

执行：

```bash
less numbers.txt
```

在 `less` 中输入：

```text
/50
```

然后按：

```text
n
q
```

---

## 3. head：查看文件开头

默认查看前 10 行：

```bash
head numbers.txt
```

查看前 5 行：

```bash
head -n 5 numbers.txt
```

### 练习

执行：

```bash
head numbers.txt
head -n 3 numbers.txt
head -n 20 numbers.txt
```

---

## 4. tail：查看文件结尾

默认查看最后 10 行：

```bash
tail numbers.txt
```

查看最后 5 行：

```bash
tail -n 5 numbers.txt
```

实时跟踪文件变化：

```bash
tail -f app.log
```

`tail -f` 常用于查看日志。

### 练习：模拟日志变化

创建日志文件：

```bash
touch app.log
```

在一个终端里执行：

```bash
tail -f app.log
```

再打开另一个终端，执行：

```bash
cd ~/linux-practice/stage-01/view-demo
echo "service started" >> app.log
echo "user login" >> app.log
echo "error: config missing" >> app.log
```

回到第一个终端，你会看到新增内容。

停止 `tail -f`：

```text
Ctrl + C
```

---

## 5. file：查看文件类型

Linux 不完全依赖扩展名判断文件类型。`file` 可以识别文件真实类型。

执行：

```bash
file words.txt
file numbers.txt
file .
```

你可能看到：

```text
words.txt: ASCII text
.: directory
```

### 练习

执行：

```bash
cd ~/linux-practice/stage-01/view-demo
mkdir sample-dir
touch empty-file
file words.txt
file empty-file
file sample-dir
```

---

## 6. stat：查看文件详细信息

`stat` 用来查看文件大小、权限、所有者、时间等详细信息。

执行：

```bash
stat words.txt
```

重点观察：

- `Size`：大小
- `Access`：权限
- `Uid`：用户 ID 和所有者
- `Gid`：组 ID 和所属组
- `Modify`：修改时间

### 练习

执行：

```bash
stat words.txt
sleep 2
echo "New line" >> words.txt
stat words.txt
```

观察文件大小和修改时间的变化。

---

## 7. 什么时候用哪个命令

| 场景 | 推荐命令 |
| --- | --- |
| 看短文件 | `cat` |
| 看长文件 | `less` |
| 看文件前几行 | `head` |
| 看文件最后几行 | `tail` |
| 跟踪日志变化 | `tail -f` |
| 判断文件类型 | `file` |
| 查看文件详细信息 | `stat` |

---

## 8. 本节小结

本节必须记住：

```bash
cat file.txt
cat -n file.txt
less file.txt
head -n 20 file.txt
tail -n 20 file.txt
tail -f app.log
file file.txt
stat file.txt
```

完成后进入下一节：`04-help-system.md`。

