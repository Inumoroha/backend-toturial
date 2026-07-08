# 第三阶段：用户、权限与文件安全教程

本阶段目标：理解 Linux 的用户、用户组和权限模型，能正确设置文件权限，能排查 `Permission denied`，并知道如何谨慎使用 `sudo`。

建议学习时间：1 周。  
建议学习方式：按文件顺序学习，每节先看命令顺序，再跟着练习。  
练习目录：`~/linux-practice/stage-03`

---

## 学习文件顺序

| 顺序 | 文件 | 主题 | 重点命令 |
| --- | --- | --- | --- |
| 1 | `01-users-and-groups.md` | 用户与用户组基础 | `whoami`、`id`、`groups`、`who` |
| 2 | `02-read-permissions.md` | 读懂权限 | `ls -l`、权限三组、文件和目录权限差异 |
| 3 | `03-chmod.md` | 修改权限 | `chmod`、数字权限、符号权限 |
| 4 | `04-owner-and-group.md` | 所有者和所属组 | `chown`、`chgrp` |
| 5 | `05-sudo-su-users.md` | sudo、su 与用户管理 | `sudo`、`su`、`useradd`、`usermod`、`passwd`、`groupadd` |
| 6 | `06-umask-special-permissions.md` | 默认权限与特殊权限 | `umask`、SUID、SGID、Sticky Bit |
| 7 | `07-stage-practice.md` | 第三阶段综合练习 | 综合使用本阶段所有命令 |

---

## 第三阶段命令学习总顺序

建议按下面顺序学习：

```text
1. whoami
2. id
3. groups
4. who
5. ls -l
6. chmod
7. chgrp
8. chown
9. sudo
10. su
11. groupadd
12. useradd
13. passwd
14. usermod
15. umask
16. stat
```

需要 `sudo` 的命令不要急着运行，先读懂用途，再在练习环境里执行。

---

## 开始前准备

创建第三阶段练习目录：

```bash
mkdir -p ~/linux-practice/stage-03/{docs,scripts,shared,private,tmp}
cd ~/linux-practice/stage-03
pwd
```

创建练习文件：

```bash
echo "public note" > docs/public.txt
echo "private note" > private/secret.txt
echo '#!/usr/bin/env bash' > scripts/hello.sh
echo 'echo "hello permission"' >> scripts/hello.sh
```

查看结构：

```bash
tree ~/linux-practice/stage-03
```

---

## 本阶段安全提醒

本阶段会接触权限和用户管理，容易因为命令写错导致系统文件权限异常。请遵守：

- 所有普通练习都在 `~/linux-practice/stage-03` 中完成。
- 不要对 `/`、`/etc`、`/usr`、`/bin` 执行递归 `chmod` 或 `chown`。
- 不要运行来源不明的 `sudo` 命令。
- 删除练习用户或练习组前，确认名字是自己创建的。
- 看到 `Permission denied` 时，先查看权限，不要第一时间乱加 `sudo`。

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 能解释 `uid`、`gid`、用户组的含义。
- 能读懂 `-rw-r--r--`、`drwxr-xr-x`。
- 能解释文件权限和目录权限的区别。
- 能用数字方式设置 `644`、`600`、`755`、`700`。
- 能用符号方式设置 `u+x`、`g+w`、`o-r`。
- 能使用 `chown` 和 `chgrp` 修改所有者和所属组。
- 能正确使用 `sudo`，知道什么时候不该用。
- 能创建练习用户和用户组。
- 能解释 `umask` 的作用。
- 对 SUID、SGID、Sticky Bit 有基础认识。

