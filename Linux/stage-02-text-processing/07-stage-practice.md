# 07：第二阶段综合练习

本节目标：综合使用第二阶段命令，完成日志分析和用户数据处理。

请在 Linux 终端中按顺序完成，不要跳步骤。

---

## 1. 本练习会用到的命令

按出现顺序：

```text
1. cd
2. cat
3. grep
4. wc
5. sort
6. uniq
7. cut
8. awk
9. sed
10. find
11. xargs
12. tee
13. >
14. >>
```

进入练习目录：

```bash
cd ~/linux-practice/stage-02
pwd
```

---

## 2. 检查练习数据

查看目录结构：

```bash
tree
```

查看用户数据：

```bash
cat data/users.csv
```

查看访问日志：

```bash
cat logs/access.log
```

查看应用日志：

```bash
cat logs/app.log
```

---

## 3. 日志基础分析

统计应用日志总行数：

```bash
wc -l logs/app.log
```

统计错误数量：

```bash
grep "ERROR" logs/app.log | wc -l
```

统计警告数量：

```bash
grep "WARN" logs/app.log | wc -l
```

提取错误行并保存：

```bash
grep "ERROR" logs/app.log > output/errors.log
cat output/errors.log
```

把警告行追加到同一个文件：

```bash
grep "WARN" logs/app.log >> output/errors.log
cat output/errors.log
```

---

## 4. 访问日志状态码分析

查看非 200 请求：

```bash
awk '$5 != 200 {print $0}' logs/access.log
```

统计每种状态码数量：

```bash
awk '{print $5}' logs/access.log | sort | uniq -c | sort -nr
```

保存状态码统计：

```bash
awk '{print $5}' logs/access.log | sort | uniq -c | sort -nr | tee output/status-count.txt
```

---

## 5. IP 访问次数分析

提取所有 IP：

```bash
awk '{print $1}' logs/access.log
```

统计 IP 出现次数：

```bash
awk '{print $1}' logs/access.log | sort | uniq -c | sort -nr
```

保存前 3 个访问 IP：

```bash
awk '{print $1}' logs/access.log | sort | uniq -c | sort -nr | head -n 3 > output/top-ip.txt
cat output/top-ip.txt
```

---

## 6. 用户数据处理

查看用户 CSV：

```bash
cat data/users.csv
```

提取姓名列：

```bash
cut -d ',' -f 2 data/users.csv
```

提取角色列并统计角色数量：

```bash
cut -d ',' -f 3 data/users.csv | tail -n +2 | sort | uniq -c | sort -nr
```

用 `awk` 找出所有 developer：

```bash
awk -F ',' '$3 == "developer" {print $0}' data/users.csv
```

只输出 developer 的姓名和城市：

```bash
awk -F ',' '$3 == "developer" {print $2, $4}' data/users.csv
```

---

## 7. sed 文本清洗

把应用日志里的 `ERROR` 替换为 `CRITICAL`：

```bash
sed 's/ERROR/CRITICAL/g' logs/app.log
```

保存替换结果：

```bash
sed 's/ERROR/CRITICAL/g' logs/app.log > output/app-critical.log
```

删除 `INFO` 行，只保留需要关注的日志：

```bash
sed '/INFO/d' logs/app.log > output/app-important.log
cat output/app-important.log
```

---

## 8. find 与 xargs 综合

查找所有 `.log` 文件：

```bash
find . -name "*.log"
```

在所有 `.log` 文件中搜索 `ERROR`：

```bash
find . -name "*.log" | xargs grep "ERROR"
```

统计所有 `.log` 文件行数：

```bash
find . -name "*.log" | xargs wc -l
```

更安全的写法：

```bash
find . -name "*.log" -print0 | xargs -0 wc -l
```

---

## 9. 生成分析报告

创建报告：

```bash
echo "# Stage 02 Report" > output/report.md
```

追加应用日志统计：

```bash
echo "## App Log" >> output/report.md
echo "Total lines:" >> output/report.md
wc -l logs/app.log >> output/report.md
echo "Errors:" >> output/report.md
grep "ERROR" logs/app.log | wc -l >> output/report.md
```

追加状态码统计：

```bash
echo "## HTTP Status Count" >> output/report.md
awk '{print $5}' logs/access.log | sort | uniq -c | sort -nr >> output/report.md
```

追加 IP 统计：

```bash
echo "## Top IP" >> output/report.md
awk '{print $1}' logs/access.log | sort | uniq -c | sort -nr >> output/report.md
```

查看报告：

```bash
cat output/report.md
```

---

## 10. 阶段验收题

请你不看答案，回答下面问题：

1. `>` 和 `>>` 有什么区别？
2. `2>` 是保存什么输出？
3. 管道 `|` 的作用是什么？
4. `grep -n`、`grep -i`、`grep -v` 分别是什么意思？
5. `find . -name "*.log"` 是做什么的？
6. 为什么 `uniq` 常常要和 `sort` 一起用？
7. `cut -d ',' -f 2` 是什么意思？
8. `tr -d` 是做什么的？
9. `sed 's/old/new/g'` 中的 `g` 是什么意思？
10. `awk '{print $1}'` 是做什么的？

---

## 11. 阶段完成标准

当你能独立完成下面任务，就可以进入第三阶段：

- [ ] 使用 `>` 和 `>>` 保存命令输出。
- [ ] 使用 `2>` 保存错误信息。
- [ ] 使用管道连接至少 3 个命令。
- [ ] 使用 `grep` 搜索日志。
- [ ] 使用 `find` 查找指定文件。
- [ ] 使用 `wc` 统计行数。
- [ ] 使用 `sort | uniq -c | sort -nr` 统计频次。
- [ ] 使用 `cut` 提取 CSV 字段。
- [ ] 使用 `sed` 替换或删除文本行。
- [ ] 使用 `awk` 按列提取和过滤数据。
- [ ] 生成一个简单的日志分析报告。

---

## 12. 建议写入学习笔记

记录下面内容：

```text
1. 我最常用的 5 条管道命令
2. grep、sed、awk 的区别
3. sort 和 uniq 为什么常一起用
4. find 和 grep 如何配合
5. 一条我自己写出来的日志分析命令
```

完成这些记录后，第二阶段就真正学完了。

