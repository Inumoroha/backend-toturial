# 01：重定向与管道

本节目标：理解标准输入、标准输出、标准错误，并学会用重定向和管道组合命令。

本节命令顺序：

```text
1. >
2. >>
3. <
4. 2>
5. |
6. tee
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-02
```

---

## 1. 三个标准流

Linux 命令通常有三个标准流：

| 名称 | 编号 | 含义 |
| --- | --- | --- |
| 标准输入 | `0` | 命令接收的数据 |
| 标准输出 | `1` | 命令正常输出的结果 |
| 标准错误 | `2` | 命令报错输出的信息 |

例如：

```bash
ls
```

正常输出是标准输出。

再看一个错误：

```bash
ls not-exist
```

报错信息是标准错误。

---

## 2. `>`：覆盖写入

`>` 把标准输出写入文件。如果文件已存在，会覆盖原内容。

执行：

```bash
echo "hello linux" > output/message.txt
cat output/message.txt
```

再次写入：

```bash
echo "new content" > output/message.txt
cat output/message.txt
```

你会发现旧内容被覆盖了。

### 练习

```bash
date > output/date.txt
cat output/date.txt
```

---

## 3. `>>`：追加写入

`>>` 把标准输出追加到文件末尾。

执行：

```bash
echo "line 1" > output/append-demo.txt
echo "line 2" >> output/append-demo.txt
echo "line 3" >> output/append-demo.txt
cat output/append-demo.txt
```

输出应该是：

```text
line 1
line 2
line 3
```

### 练习

```bash
whoami >> output/date.txt
pwd >> output/date.txt
cat output/date.txt
```

---

## 4. `<`：输入重定向

`<` 把文件内容作为命令的标准输入。

例如统计文件行数：

```bash
wc -l < data/users.csv
```

对比：

```bash
wc -l data/users.csv
```

区别：

- `wc -l data/users.csv` 会显示文件名。
- `wc -l < data/users.csv` 只显示行数。

---

## 5. `2>`：保存错误信息

`2>` 把标准错误写入文件。

执行一个会失败的命令：

```bash
ls not-exist 2> output/error.txt
cat output/error.txt
```

把正常输出和错误输出分别保存：

```bash
ls data not-exist > output/stdout.txt 2> output/stderr.txt
cat output/stdout.txt
cat output/stderr.txt
```

把正常输出和错误输出合并保存：

```bash
ls data not-exist > output/all.txt 2>&1
cat output/all.txt
```

---

## 6. `|`：管道

管道把前一个命令的输出交给后一个命令处理。

查看日志中包含 `ERROR` 的行：

```bash
cat logs/app.log | grep "ERROR"
```

统计错误行数：

```bash
cat logs/app.log | grep "ERROR" | wc -l
```

从访问日志中提取 404 行：

```bash
cat logs/access.log | grep "404"
```

### 小提醒

很多命令可以直接接文件名，例如：

```bash
grep "ERROR" logs/app.log
```

但学习管道时，可以先写成：

```bash
cat logs/app.log | grep "ERROR"
```

这样更容易理解数据流动方向。

---

## 7. tee：一边显示，一边写入文件

`tee` 会把输入同时输出到屏幕和文件。

执行：

```bash
grep "ERROR" logs/app.log | tee output/errors.txt
```

查看保存结果：

```bash
cat output/errors.txt
```

追加写入：

```bash
grep "WARN" logs/app.log | tee -a output/errors.txt
cat output/errors.txt
```

---

## 8. 本节小结

必须记住：

```bash
echo "text" > file.txt
echo "more" >> file.txt
wc -l < file.txt
ls not-exist 2> error.txt
command1 | command2
command1 | tee result.txt
```

理解这句话就算入门了：

```text
重定向负责把输出保存到文件，管道负责把输出交给下一个命令。
```

完成后进入下一节：`02-grep-search.md`。

