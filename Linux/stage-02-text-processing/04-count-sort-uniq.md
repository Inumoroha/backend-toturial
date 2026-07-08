# 04：wc、sort 与 uniq

本节目标：学会统计文本、排序文本、去重文本，并统计重复次数。

本节命令顺序：

```text
1. wc
2. sort
3. uniq
4. sort | uniq
5. sort | uniq -c | sort -nr
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-02
```

---

## 1. wc：统计行数、单词数、字节数

统计文件：

```bash
wc logs/app.log
```

只统计行数：

```bash
wc -l logs/app.log
```

只统计单词数：

```bash
wc -w logs/app.log
```

只统计字节数：

```bash
wc -c logs/app.log
```

常用参数：

| 参数 | 含义 |
| --- | --- |
| `-l` | 行数 |
| `-w` | 单词数 |
| `-c` | 字节数 |
| `-m` | 字符数 |

### 练习

```bash
wc -l data/users.csv
wc -l logs/access.log
grep "ERROR" logs/app.log | wc -l
```

---

## 2. sort：排序

创建练习文件：

```bash
cat > tmp/names.txt <<'EOF'
Bob
Alice
Eve
Carol
David
Alice
Bob
EOF
```

按字母排序：

```bash
sort tmp/names.txt
```

倒序排序：

```bash
sort -r tmp/names.txt
```

---

## 3. 数字排序

创建数字文件：

```bash
cat > tmp/scores.txt <<'EOF'
10
2
30
25
100
EOF
```

普通排序：

```bash
sort tmp/scores.txt
```

数字排序：

```bash
sort -n tmp/scores.txt
```

数字倒序：

```bash
sort -nr tmp/scores.txt
```

注意：不加 `-n` 时，`100` 可能排在 `2` 前面，因为它按字符串排序。

---

## 4. uniq：去重

`uniq` 只能去除相邻的重复行，所以通常先 `sort`。

直接去重：

```bash
uniq tmp/names.txt
```

先排序再去重：

```bash
sort tmp/names.txt | uniq
```

统计重复次数：

```bash
sort tmp/names.txt | uniq -c
```

按次数倒序：

```bash
sort tmp/names.txt | uniq -c | sort -nr
```

---

## 5. 分析访问日志 IP

访问日志第一列是 IP。

先查看日志：

```bash
cat logs/access.log
```

提取第一列：

```bash
awk '{print $1}' logs/access.log
```

统计每个 IP 出现次数：

```bash
awk '{print $1}' logs/access.log | sort | uniq -c
```

按次数倒序：

```bash
awk '{print $1}' logs/access.log | sort | uniq -c | sort -nr
```

这是一条很常见的日志分析命令。

---

## 6. 本节小结

必须记住：

```bash
wc -l file
sort file
sort -n file
sort -nr file
sort file | uniq
sort file | uniq -c
sort file | uniq -c | sort -nr
awk '{print $1}' access.log | sort | uniq -c | sort -nr
```

完成后进入下一节：`05-cut-tr.md`。

