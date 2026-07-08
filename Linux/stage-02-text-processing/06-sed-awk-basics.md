# 06：sed 与 awk 入门

本节目标：掌握 `sed` 的基础替换、删除、打印，以及 `awk` 的字段提取、过滤和简单统计。

本节命令顺序：

```text
1. sed 's/old/new/'
2. sed 's/old/new/g'
3. sed -n '/pattern/p'
4. sed '/pattern/d'
5. awk '{print $1}'
6. awk -F
7. awk 'condition'
8. awk simple count
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-02
```

---

## 1. sed 基础替换

把每行第一个 `INFO` 替换成 `NOTICE`：

```bash
sed 's/INFO/NOTICE/' logs/app.log
```

注意：这不会修改原文件，只是输出修改后的结果。

保存到新文件：

```bash
sed 's/INFO/NOTICE/' logs/app.log > output/app-notice.log
cat output/app-notice.log
```

---

## 2. sed 全局替换

默认只替换每一行第一个匹配项。

创建练习文本：

```bash
echo "error error error" > tmp/repeat.txt
```

只替换第一个：

```bash
sed 's/error/ERROR/' tmp/repeat.txt
```

替换所有：

```bash
sed 's/error/ERROR/g' tmp/repeat.txt
```

`g` 表示 global，也就是一行内全部替换。

---

## 3. sed 打印匹配行

只打印包含 `ERROR` 的行：

```bash
sed -n '/ERROR/p' logs/app.log
```

说明：

| 部分 | 含义 |
| --- | --- |
| `-n` | 不自动打印所有行 |
| `/ERROR/` | 匹配包含 ERROR 的行 |
| `p` | 打印匹配行 |

这个命令效果类似：

```bash
grep "ERROR" logs/app.log
```

---

## 4. sed 删除匹配行

删除包含 `INFO` 的行：

```bash
sed '/INFO/d' logs/app.log
```

删除空行：

```bash
sed '/^$/d' logs/app.log
```

保存删除后的结果：

```bash
sed '/INFO/d' logs/app.log > output/no-info.log
cat output/no-info.log
```

---

## 5. awk 提取字段

`awk` 默认按空格或连续空白分隔字段。

提取访问日志第 1 列 IP：

```bash
awk '{print $1}' logs/access.log
```

提取第 3 列请求方法：

```bash
awk '{print $3}' logs/access.log
```

提取第 4 列路径和第 5 列状态码：

```bash
awk '{print $4, $5}' logs/access.log
```

---

## 6. awk 指定分隔符

CSV 文件用逗号分隔，需要用 `-F` 指定分隔符。

提取姓名：

```bash
awk -F ',' '{print $2}' data/users.csv
```

提取姓名和角色：

```bash
awk -F ',' '{print $2, $3}' data/users.csv
```

输出时自定义格式：

```bash
awk -F ',' '{print "name=" $2 ", role=" $3}' data/users.csv
```

---

## 7. awk 条件过滤

找出角色是 developer 的用户：

```bash
awk -F ',' '$3 == "developer" {print $0}' data/users.csv
```

只打印姓名：

```bash
awk -F ',' '$3 == "developer" {print $2}' data/users.csv
```

找出状态码是 404 的访问：

```bash
awk '$5 == 404 {print $0}' logs/access.log
```

找出非 200 的访问：

```bash
awk '$5 != 200 {print $0}' logs/access.log
```

---

## 8. awk 简单统计

统计访问日志中每个 IP 的出现次数：

```bash
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' logs/access.log
```

结合排序：

```bash
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' logs/access.log | sort -nr
```

统计每种状态码数量：

```bash
awk '{count[$5]++} END {for (code in count) print count[code], code}' logs/access.log | sort -nr
```

现在不需要完全掌握 `awk` 的所有语法，先记住它适合做字段提取和按列过滤。

---

## 9. sed 与 awk 怎么选

| 任务 | 推荐工具 |
| --- | --- |
| 搜索行 | `grep` |
| 简单替换 | `sed` |
| 删除匹配行 | `sed` |
| 提取第几列 | `awk` 或 `cut` |
| 按列判断过滤 | `awk` |
| 简单统计 | `awk` |

---

## 10. 本节小结

必须记住：

```bash
sed 's/old/new/' file
sed 's/old/new/g' file
sed -n '/pattern/p' file
sed '/pattern/d' file
awk '{print $1}' file
awk -F ',' '{print $2}' file.csv
awk '$5 == 404 {print $0}' access.log
awk '{count[$1]++} END {for (k in count) print count[k], k}' file
```

完成后进入下一节：`07-stage-practice.md`。

