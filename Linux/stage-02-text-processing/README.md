# 第二阶段：文本处理与管道教程

本阶段目标：掌握 Linux 命令行中最重要的组合能力：重定向、管道、搜索、统计、排序、字段提取和基础文本替换。

建议学习时间：2 周。  
建议学习方式：按文件顺序学习，每节先看命令顺序，再跟着练习。  
练习目录：`~/linux-practice/stage-02`

---

## 学习文件顺序

| 顺序 | 文件 | 主题 | 重点命令 |
| --- | --- | --- | --- |
| 1 | `01-redirection-and-pipe.md` | 标准输入输出、重定向、管道 | `>`、`>>`、`<`、`2>`、`|`、`tee` |
| 2 | `02-grep-search.md` | 文本搜索 | `grep`、`egrep`、`fgrep` |
| 3 | `03-find-and-xargs.md` | 文件查找与批量处理 | `find`、`xargs` |
| 4 | `04-count-sort-uniq.md` | 统计、排序、去重 | `wc`、`sort`、`uniq` |
| 5 | `05-cut-tr.md` | 字段提取与字符转换 | `cut`、`tr` |
| 6 | `06-sed-awk-basics.md` | sed 和 awk 入门 | `sed`、`awk` |
| 7 | `07-stage-practice.md` | 第二阶段综合练习 | 综合使用本阶段所有命令 |

---

## 第二阶段命令学习总顺序

建议严格按下面顺序学习：

```text
1. >
2. >>
3. <
4. 2>
5. |
6. tee
7. grep
8. find
9. xargs
10. wc
11. sort
12. uniq
13. cut
14. tr
15. sed
16. awk
```

学习重点不是“背命令”，而是学会组合：

```bash
cat file.txt | grep "keyword" | sort | uniq -c | sort -nr
```

这类组合命令是 Linux 文本处理的核心能力。

---

## 开始前准备

创建第二阶段练习目录：

```bash
mkdir -p ~/linux-practice/stage-02/{data,logs,output,tmp}
cd ~/linux-practice/stage-02
pwd
```

后续练习都在这个目录中完成。

---

## 准备练习数据

进入练习目录：

```bash
cd ~/linux-practice/stage-02
```

创建用户数据：

```bash
cat > data/users.csv <<'EOF'
id,name,role,city
1,Alice,admin,Beijing
2,Bob,developer,Shanghai
3,Carol,developer,Beijing
4,David,ops,Shenzhen
5,Eve,admin,Hangzhou
6,Frank,developer,Shanghai
EOF
```

创建访问日志：

```bash
cat > logs/access.log <<'EOF'
192.168.1.10 - GET /index.html 200
192.168.1.11 - GET /login 200
192.168.1.12 - POST /login 403
192.168.1.10 - GET /dashboard 200
192.168.1.13 - GET /missing 404
192.168.1.11 - GET /index.html 200
192.168.1.14 - GET /missing 404
192.168.1.12 - POST /login 403
192.168.1.10 - GET /settings 500
EOF
```

创建错误日志：

```bash
cat > logs/app.log <<'EOF'
INFO 2026-07-02 10:00:01 service started
INFO 2026-07-02 10:01:12 user Alice login
WARN 2026-07-02 10:02:33 disk usage high
ERROR 2026-07-02 10:03:44 database timeout
INFO 2026-07-02 10:04:05 user Bob login
ERROR 2026-07-02 10:05:16 config missing
WARN 2026-07-02 10:06:27 memory usage high
INFO 2026-07-02 10:07:38 service stopped
EOF
```

检查数据：

```bash
tree
cat data/users.csv
cat logs/access.log
cat logs/app.log
```

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 能解释标准输入、标准输出、标准错误。
- 能使用 `>`、`>>`、`2>` 保存输出和错误。
- 能使用管道把多个命令连接起来。
- 能用 `grep` 搜索文本。
- 能用 `find` 查找文件。
- 能用 `wc` 统计行数、单词数、字符数。
- 能用 `sort` 和 `uniq` 排序、去重、统计频次。
- 能用 `cut` 提取字段。
- 能用 `tr` 做简单字符转换。
- 能用 `sed` 做基础替换和删除。
- 能用 `awk` 提取字段、过滤行、做简单统计。

