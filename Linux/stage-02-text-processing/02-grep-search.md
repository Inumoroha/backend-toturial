# 02：grep 文本搜索

本节目标：学会在文本中搜索关键词、忽略大小写、显示行号、反向匹配和递归搜索。

本节命令顺序：

```text
1. grep "keyword" file
2. grep -n
3. grep -i
4. grep -v
5. grep -r
6. grep -E
7. grep -F
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-02
```

---

## 1. 基础搜索

搜索包含 `ERROR` 的行：

```bash
grep "ERROR" logs/app.log
```

搜索包含 `login` 的行：

```bash
grep "login" logs/app.log
```

搜索访问日志中的 `404`：

```bash
grep "404" logs/access.log
```

---

## 2. `-n`：显示行号

显示匹配行的行号：

```bash
grep -n "ERROR" logs/app.log
```

行号在排查日志时非常有用。

### 练习

```bash
grep -n "WARN" logs/app.log
grep -n "403" logs/access.log
```

---

## 3. `-i`：忽略大小写

忽略大小写搜索：

```bash
grep -i "error" logs/app.log
```

这会同时匹配 `error`、`ERROR`、`Error`。

### 练习

```bash
grep -i "info" logs/app.log
grep -i "alice" logs/app.log
```

---

## 4. `-v`：反向匹配

显示不包含某个关键词的行：

```bash
grep -v "INFO" logs/app.log
```

显示不是 200 的访问记录：

```bash
grep -v "200" logs/access.log
```

### 练习

```bash
grep -v "developer" data/users.csv
```

---

## 5. `-r`：递归搜索目录

在整个目录中搜索：

```bash
grep -r "ERROR" .
```

在日志目录中搜索：

```bash
grep -rn "login" logs
```

常用组合：

```bash
grep -rni "error" logs
```

含义：

| 参数 | 含义 |
| --- | --- |
| `-r` | 递归搜索目录 |
| `-n` | 显示行号 |
| `-i` | 忽略大小写 |

---

## 6. `-E`：扩展正则

`grep -E` 支持更方便的正则表达式，也可以写成 `egrep`。

搜索 `ERROR` 或 `WARN`：

```bash
grep -E "ERROR|WARN" logs/app.log
```

搜索状态码 403 或 404：

```bash
grep -E "403|404" logs/access.log
```

### 常见正则符号

| 符号 | 含义 |
| --- | --- |
| `.` | 任意单个字符 |
| `*` | 前一个字符出现 0 次或多次 |
| `+` | 前一个字符出现 1 次或多次 |
| `?` | 前一个字符出现 0 次或 1 次 |
| `|` | 或 |
| `^` | 行首 |
| `$` | 行尾 |

### 练习

匹配以 `INFO` 开头的行：

```bash
grep -E "^INFO" logs/app.log
```

匹配以 `500` 结尾的行：

```bash
grep -E "500$" logs/access.log
```

---

## 7. `-F`：固定字符串搜索

`grep -F` 把搜索内容当成普通字符串，不当成正则。

例如搜索包含点号的 IP：

```bash
grep -F "192.168.1.10" logs/access.log
```

也可以写成：

```bash
fgrep "192.168.1.10" logs/access.log
```

新手遇到特殊符号时，`grep -F` 很有用。

---

## 8. grep 与管道组合

统计错误数量：

```bash
grep "ERROR" logs/app.log | wc -l
```

保存错误日志：

```bash
grep "ERROR" logs/app.log > output/errors-only.log
```

搜索非正常状态码：

```bash
grep -E "403|404|500" logs/access.log
```

---

## 9. 本节小结

必须记住：

```bash
grep "keyword" file
grep -n "keyword" file
grep -i "keyword" file
grep -v "keyword" file
grep -rn "keyword" dir
grep -E "A|B" file
grep -F "plain.string" file
```

完成后进入下一节：`03-find-and-xargs.md`。

