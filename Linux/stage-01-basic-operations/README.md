# 第一阶段：Linux 基础操作教程

本阶段目标：理解 Linux 文件系统结构，掌握文件和目录的基础操作，学会查看文本文件，并能使用帮助系统自学命令。

建议学习时间：1 到 2 周。  
建议学习方式：按文件顺序学习，每节完成后再进入下一节。  
练习目录：`~/linux-practice/stage-01`

---

## 学习文件顺序

请按下面顺序阅读和练习：

| 顺序 | 文件 | 主题 | 重点命令 |
| --- | --- | --- | --- |
| 1 | `01-paths-and-directories.md` | 路径、目录、文件系统 | `pwd`、`ls`、`cd`、`tree` |
| 2 | `02-create-copy-move-delete.md` | 创建、复制、移动、删除 | `mkdir`、`rmdir`、`touch`、`cp`、`mv`、`rm` |
| 3 | `03-view-files.md` | 查看文件内容和文件信息 | `cat`、`less`、`head`、`tail`、`file`、`stat` |
| 4 | `04-help-system.md` | 命令帮助系统 | `--help`、`man`、`type`、`which`、`whereis` |
| 5 | `05-stage-practice.md` | 第一阶段综合练习 | 综合使用本阶段所有命令 |

---

## 第一阶段命令学习总顺序

不要一开始就背全部命令。建议严格按下面顺序学：

```text
1. pwd
2. ls
3. cd
4. tree
5. mkdir
6. rmdir
7. touch
8. cp
9. mv
10. rm
11. cat
12. less
13. head
14. tail
15. file
16. stat
17. command --help
18. man
19. type
20. which
21. whereis
```

学习原则：

- 先知道命令解决什么问题。
- 再运行最简单的命令。
- 然后增加一个参数。
- 最后做一个小练习。

---

## 开始前准备

进入 Linux 终端，先创建第一阶段练习目录：

```bash
mkdir -p ~/linux-practice/stage-01
cd ~/linux-practice/stage-01
pwd
```

如果输出类似下面这样，说明准备好了：

```text
/home/你的用户名/linux-practice/stage-01
```

后续所有危险操作，比如 `rm`，都只在这个练习目录里做。

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 能解释 `/`、`~`、`.`、`..` 的含义。
- 能区分绝对路径和相对路径。
- 能用命令创建、复制、移动、重命名、删除文件和目录。
- 能查看短文件、长文件、文件开头、文件结尾。
- 能查看文件类型和文件详细信息。
- 遇到陌生命令时，能用 `--help` 或 `man` 自己查用法。
- 能独立完成 `05-stage-practice.md` 里的综合练习。

