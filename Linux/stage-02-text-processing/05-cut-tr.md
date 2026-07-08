# 05：cut 与 tr

本节目标：学会按分隔符提取字段，并进行简单字符转换、删除和压缩。

本节命令顺序：

```text
1. cut -d -f
2. cut -c
3. tr
4. tr -d
5. tr -s
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-02
```

---

## 1. cut：按字段提取

查看 CSV 文件：

```bash
cat data/users.csv
```

CSV 使用逗号分隔字段。

提取第 2 列，也就是姓名：

```bash
cut -d ',' -f 2 data/users.csv
```

提取第 2 列和第 3 列：

```bash
cut -d ',' -f 2,3 data/users.csv
```

提取第 2 到第 4 列：

```bash
cut -d ',' -f 2-4 data/users.csv
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-d` | 指定分隔符 |
| `-f` | 指定字段 |

---

## 2. cut：按字符位置提取

按字符提取前 5 个字符：

```bash
cut -c 1-5 logs/app.log
```

提取第 1 个字符：

```bash
cut -c 1 logs/app.log
```

按字符提取适合固定格式文本，不适合字段长度变化很大的数据。

---

## 3. 提取日志字段

访问日志用空格分隔。

提取第 1 列 IP：

```bash
cut -d ' ' -f 1 logs/access.log
```

提取第 4 列路径：

```bash
cut -d ' ' -f 4 logs/access.log
```

提取第 5 列状态码：

```bash
cut -d ' ' -f 5 logs/access.log
```

如果文本中有多个连续空格，`cut` 可能不如 `awk` 稳定。这个差异后面会看到。

---

## 4. tr：字符转换

把小写变大写：

```bash
echo "hello linux" | tr 'a-z' 'A-Z'
```

把大写变小写：

```bash
echo "HELLO LINUX" | tr 'A-Z' 'a-z'
```

把逗号换成空格：

```bash
cat data/users.csv | tr ',' ' '
```

---

## 5. tr -d：删除字符

删除逗号：

```bash
echo "a,b,c" | tr -d ','
```

删除数字：

```bash
echo "abc123def456" | tr -d '0-9'
```

删除换行，把多行变成一行：

```bash
cat data/users.csv | tr -d '\n'
```

---

## 6. tr -s：压缩重复字符

压缩多个空格为一个空格：

```bash
echo "Linux     is     good" | tr -s ' '
```

压缩重复冒号：

```bash
echo "a:::b::::c" | tr -s ':'
```

---

## 7. cut 与管道组合

提取用户角色，并统计每种角色数量：

```bash
cut -d ',' -f 3 data/users.csv | tail -n +2 | sort | uniq -c | sort -nr
```

说明：

| 部分 | 作用 |
| --- | --- |
| `cut -d ',' -f 3` | 提取第 3 列 |
| `tail -n +2` | 从第 2 行开始，跳过表头 |
| `sort` | 排序 |
| `uniq -c` | 统计次数 |
| `sort -nr` | 按数字倒序 |

---

## 8. 本节小结

必须记住：

```bash
cut -d ',' -f 2 file.csv
cut -d ',' -f 2,3 file.csv
cut -c 1-5 file.txt
echo "abc" | tr 'a-z' 'A-Z'
echo "a,b,c" | tr -d ','
echo "a     b" | tr -s ' '
```

完成后进入下一节：`06-sed-awk-basics.md`。

