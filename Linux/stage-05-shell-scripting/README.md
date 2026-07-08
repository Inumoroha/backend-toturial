# 第五阶段：Shell 脚本入门教程

本阶段目标：能编写简单 Bash 脚本，把重复命令自动化；理解变量、参数、条件、循环、函数、退出状态和调试方法。

建议学习时间：2 周。  
建议学习方式：按文件顺序学习，每节先看“本节语法/命令顺序”，再动手写脚本。  
练习目录：`~/linux-practice/stage-05`

---

## 学习文件顺序

| 顺序 | 文件 | 主题 | 重点 |
| --- | --- | --- | --- |
| 1 | `01-first-script.md` | 第一个 Shell 脚本 | Shebang、执行脚本、执行权限 |
| 2 | `02-variables-and-quotes.md` | 变量与引用 | 变量、双引号、单引号、命令替换 |
| 3 | `03-parameters-and-exit-status.md` | 参数与退出状态 | `$1`、`$@`、`$#`、`$?`、`exit` |
| 4 | `04-conditionals.md` | 条件判断 | `if`、`test`、文件判断、字符串判断、数字判断 |
| 5 | `05-loops.md` | 循环 | `for`、`while`、`break`、`continue` |
| 6 | `06-functions-debugging.md` | 函数与调试 | 函数、局部变量、`bash -n`、`bash -x` |
| 7 | `07-practical-scripts.md` | 实用脚本 | 备份、服务检查、日志统计 |
| 8 | `08-stage-practice.md` | 第五阶段综合练习 | 完成一个个人脚本工具箱 |

---

## 第五阶段语法和命令学习总顺序

建议按下面顺序学习：

```text
1. #!/usr/bin/env bash
2. echo
3. chmod +x
4. ./script.sh
5. bash script.sh
6. variable=value
7. "$variable"
8. 'text'
9. "$(command)"
10. $1 $2
11. "$@"
12. $#
13. $?
14. exit
15. if
16. [ condition ]
17. [[ condition ]]
18. for
19. while
20. function
21. local
22. bash -n
23. bash -x
24. set -euo pipefail
```

---

## 开始前准备

创建第五阶段练习目录：

```bash
mkdir -p ~/linux-practice/stage-05/{scripts,data,logs,backup,output,tmp}
cd ~/linux-practice/stage-05
pwd
```

创建练习日志：

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

创建练习数据：

```bash
cat > data/users.txt <<'EOF'
alice
bob
carol
david
eve
EOF
```

---

## 本阶段安全提醒

Shell 脚本很强大，也很容易批量造成错误。请遵守：

- 所有练习脚本都放在 `~/linux-practice/stage-05/scripts`。
- 先用 `echo` 打印将要执行的操作，再真正执行。
- 脚本里使用变量时尽量加双引号，例如 `"$file"`。
- 删除文件前先确认路径，避免写出空变量导致误删。
- 暂时不要写递归删除系统目录的脚本。

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 能写出带 Shebang 的 Bash 脚本。
- 能用 `chmod +x` 添加执行权限。
- 能理解变量赋值和引用。
- 能正确使用 `"$@"` 处理脚本参数。
- 能用 `$?` 和 `exit` 判断命令成功失败。
- 能写 `if` 条件判断。
- 能写 `for` 和 `while` 循环。
- 能写简单函数。
- 能用 `bash -n` 和 `bash -x` 检查脚本。
- 能写出备份脚本、服务检查脚本、日志统计脚本。

