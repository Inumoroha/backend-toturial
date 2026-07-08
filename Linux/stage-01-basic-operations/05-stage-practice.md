# 05：第一阶段综合练习

本节目标：把第一阶段学过的命令串起来，完成一个完整的文件管理练习。

请在 Linux 终端中按顺序完成，不要跳步骤。

---

## 1. 本练习会用到的命令

按出现顺序：

```text
1. cd
2. pwd
3. mkdir
4. ls
5. touch
6. cp
7. mv
8. cat
9. head
10. tail
11. file
12. stat
13. tree
14. rm
15. man
```

---

## 2. 创建项目结构

进入第一阶段练习目录：

```bash
cd ~/linux-practice/stage-01
pwd
```

创建综合练习目录：

```bash
mkdir -p final-lab/{docs,logs,scripts,backup,tmp}
```

查看结构：

```bash
tree final-lab
```

预期结构：

```text
final-lab
├── backup
├── docs
├── logs
├── scripts
└── tmp
```

---

## 3. 创建文件

进入 `docs` 目录：

```bash
cd ~/linux-practice/stage-01/final-lab/docs
pwd
```

创建 3 个笔记文件：

```bash
touch linux-basics.md commands.md mistakes.md
```

查看文件：

```bash
ls -lah
```

写入一些内容：

```bash
printf "pwd: show current directory\nls: list files\ncd: change directory\n" > commands.md
printf "Do not run rm carelessly.\nCheck pwd before deleting files.\n" > mistakes.md
printf "Linux filesystem starts from root directory: /\nHome directory is represented by ~\n" > linux-basics.md
```

查看内容：

```bash
cat commands.md
cat mistakes.md
cat linux-basics.md
```

---

## 4. 复制文件

把笔记复制到备份目录：

```bash
cp -iv commands.md ../backup/
cp -iv mistakes.md ../backup/
cp -iv linux-basics.md ../backup/
```

查看备份目录：

```bash
ls -lah ../backup
```

复制整个 `docs` 目录：

```bash
cd ~/linux-practice/stage-01/final-lab
cp -r docs docs-copy
tree -L 2
```

---

## 5. 移动和重命名文件

进入临时目录：

```bash
cd ~/linux-practice/stage-01/final-lab/tmp
```

创建临时文件：

```bash
touch old-name.txt
ls -lah
```

重命名：

```bash
mv -iv old-name.txt better-name.txt
ls -lah
```

移动到 `docs` 目录：

```bash
mv -iv better-name.txt ../docs/
ls -lah
ls -lah ../docs
```

---

## 6. 查看文件内容

进入 `docs`：

```bash
cd ~/linux-practice/stage-01/final-lab/docs
```

查看完整内容：

```bash
cat commands.md
```

查看前 2 行：

```bash
head -n 2 commands.md
```

查看最后 2 行：

```bash
tail -n 2 commands.md
```

用分页方式查看：

```bash
less commands.md
```

在 `less` 中按 `q` 退出。

---

## 7. 查看文件类型和状态

查看文件类型：

```bash
file commands.md
file ../docs
```

查看文件详细信息：

```bash
stat commands.md
```

重点看：

- 文件大小
- 权限
- 所有者
- 修改时间

---

## 8. 模拟日志文件

进入日志目录：

```bash
cd ~/linux-practice/stage-01/final-lab/logs
```

创建日志：

```bash
printf "INFO service started\nINFO user login\nWARN disk usage high\nERROR config missing\n" > app.log
```

查看完整日志：

```bash
cat app.log
```

查看最后 2 行：

```bash
tail -n 2 app.log
```

追加日志：

```bash
echo "INFO service stopped" >> app.log
tail -n 3 app.log
```

---

## 9. 删除临时内容

删除前先确认位置：

```bash
cd ~/linux-practice/stage-01/final-lab
pwd
ls -lah
```

删除复制出来的目录：

```bash
rm -rv docs-copy
```

删除临时目录中的内容：

```bash
rm -rv tmp
```

再次查看结构：

```bash
tree -L 2
```

---

## 10. 使用帮助系统复盘

查看 `rm` 帮助：

```bash
rm --help
```

查看 `cp` 手册：

```bash
man cp
```

在 `man cp` 中搜索：

```text
/-r
```

按 `q` 退出。

查看命令来源：

```bash
type cd
type ls
which ls
whereis ls
```

---

## 11. 阶段验收题

请你不看答案，回答下面问题：

1. `pwd` 是做什么的？
2. `ls -lah` 中 `-l`、`-a`、`-h` 分别是什么意思？
3. `/`、`~`、`.`、`..` 分别是什么意思？
4. 创建多级目录应该用什么命令？
5. 复制目录时 `cp` 需要加什么参数？
6. `mv` 的两个常见用途是什么？
7. 为什么新手删除文件时建议用 `rm -i`？
8. 查看长文件应该用 `cat` 还是 `less`？
9. 实时查看日志应该用什么命令？
10. 忘记命令参数时应该先查什么？

---

## 12. 阶段完成标准

当你能独立完成下面操作，就可以进入第二阶段：

- [ ] 创建一个多级目录结构。
- [ ] 创建多个文件。
- [ ] 复制文件和目录。
- [ ] 移动和重命名文件。
- [ ] 删除文件和目录，并知道删除前如何确认。
- [ ] 用 `cat`、`less`、`head`、`tail` 查看文件。
- [ ] 用 `file` 和 `stat` 查看文件信息。
- [ ] 用 `--help` 和 `man` 查询命令帮助。
- [ ] 能说清楚绝对路径和相对路径的区别。

---

## 13. 建议写入学习笔记

在你的笔记中记录：

```text
1. 本阶段最常用的 10 个命令
2. 我最容易混淆的 3 个概念
3. 我遇到的 3 个错误
4. 每个错误是怎么解决的
5. rm 命令的安全使用规则
```

完成这些记录后，第一阶段就真正学完了。

