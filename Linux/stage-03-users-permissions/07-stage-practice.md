# 07：第三阶段综合练习

本节目标：综合使用第三阶段命令，完成一个“共享目录 + 私密目录 + 脚本权限 + 练习用户”的权限实验。

请在 Linux 终端中按顺序完成。带 `sudo` 的命令会修改系统用户和组，只建议在 WSL、虚拟机或测试环境中运行。

---

## 1. 本练习会用到的命令

按出现顺序：

```text
1. cd
2. pwd
3. whoami
4. id
5. groups
6. mkdir
7. touch
8. ls -l
9. chmod
10. chgrp
11. sudo groupadd
12. sudo useradd
13. sudo passwd
14. sudo usermod
15. sudo chown
16. su
17. umask
18. stat
```

---

## 2. 初始化练习环境

进入练习目录：

```bash
mkdir -p ~/linux-practice/stage-03/final-lab
cd ~/linux-practice/stage-03/final-lab
pwd
```

查看当前身份：

```bash
whoami
id
groups
```

创建目录结构：

```bash
mkdir -p project/{public,private,scripts,shared}
```

创建文件：

```bash
echo "project readme" > project/public/README.md
echo "secret token" > project/private/token.txt
echo '#!/usr/bin/env bash' > project/scripts/run.sh
echo 'echo "project script running"' >> project/scripts/run.sh
```

查看：

```bash
tree project
ls -l project
```

---

## 3. 设置普通文件和私密文件权限

普通说明文件设置为 `644`：

```bash
chmod 644 project/public/README.md
ls -l project/public/README.md
```

私密文件设置为 `600`：

```bash
chmod 600 project/private/token.txt
ls -l project/private/token.txt
```

私密目录设置为 `700`：

```bash
chmod 700 project/private
ls -ld project/private
```

脚本设置为可执行：

```bash
chmod 755 project/scripts/run.sh
ls -l project/scripts/run.sh
./project/scripts/run.sh
```

---

## 4. 使用符号方式调整权限

移除其他人对 README 的读权限：

```bash
chmod o-r project/public/README.md
ls -l project/public/README.md
```

恢复其他人的读权限：

```bash
chmod o+r project/public/README.md
ls -l project/public/README.md
```

给共享目录的组添加写权限：

```bash
chmod g+w project/shared
ls -ld project/shared
```

---

## 5. 创建练习组和练习用户

创建练习组：

```bash
sudo groupadd projectteam
```

如果提示已存在，可以继续使用这个组，或者改名为 `projectteam2`。

创建练习用户：

```bash
sudo useradd -m -s /bin/bash alice01
```

设置密码：

```bash
sudo passwd alice01
```

把用户加入项目组：

```bash
sudo usermod -aG projectteam alice01
```

查看：

```bash
id alice01
groups alice01
getent group projectteam
```

---

## 6. 设置共享目录所属组

把共享目录所属组改为 `projectteam`：

```bash
sudo chown :"projectteam" project/shared
```

设置组可写，并设置 SGID：

```bash
chmod 2775 project/shared
ls -ld project/shared
```

你应该看到类似：

```text
drwxrwsr-x
```

其中 `s` 表示 SGID，新文件会继承目录所属组。

---

## 7. 切换用户测试权限

切换到练习用户：

```bash
su - alice01
```

进入项目目录。请把下面路径中的 `你的用户名` 改成真实用户名：

```bash
cd /home/你的用户名/linux-practice/stage-03/final-lab
```

查看身份：

```bash
whoami
id
```

尝试读取公开文件：

```bash
cat project/public/README.md
```

尝试读取私密文件：

```bash
cat project/private/token.txt
```

这个命令大概率会失败，出现：

```text
Permission denied
```

尝试在共享目录创建文件：

```bash
echo "hello from alice01" > project/shared/alice-note.txt
ls -l project/shared
```

退出练习用户：

```bash
exit
```

---

## 8. 回到原用户检查结果

确认身份：

```bash
whoami
```

回到实验目录：

```bash
cd ~/linux-practice/stage-03/final-lab
```

查看共享目录：

```bash
ls -ld project/shared
ls -l project/shared
```

查看详细信息：

```bash
stat project/shared/alice-note.txt
```

观察文件所有者和所属组。

---

## 9. umask 实验

查看当前默认权限掩码：

```bash
umask
```

创建默认文件和目录：

```bash
mkdir -p project/umask-demo
cd project/umask-demo
touch normal-file.txt
mkdir normal-dir
ls -l
ls -ld normal-dir
```

临时改成更私密的默认权限：

```bash
umask 077
touch private-file.txt
mkdir private-dir
ls -l
ls -ld private-dir
```

恢复常见设置：

```bash
umask 022
```

---

## 10. 排查 Permission denied

当看到：

```text
Permission denied
```

按这个顺序排查：

```bash
whoami
id
pwd
ls -ld 目标目录
ls -l 目标文件
stat 目标文件
```

判断：

```text
1. 我是不是文件所有者？
2. 我是否属于文件所属组？
3. 如果都不是，其他人权限够不够？
4. 如果是目录，我有没有 x 权限进入？
5. 是否真的需要 sudo？
```

---

## 11. 清理练习用户和组

如果你不再需要练习用户：

```bash
sudo userdel -r alice01
```

如果你不再需要练习组：

```bash
sudo groupdel projectteam
```

如果提示组仍被使用，先确认没有用户还属于这个组：

```bash
getent group projectteam
```

---

## 12. 阶段验收题

请你不看答案，回答下面问题：

1. `uid` 和 `gid` 分别是什么意思？
2. `-rw-r--r--` 表示什么权限？
3. `drwx------` 表示什么权限？
4. 文件的 `x` 和目录的 `x` 有什么区别？
5. `chmod 755 script.sh` 的含义是什么？
6. `chmod 600 secret.txt` 适合什么场景？
7. `chown` 和 `chmod` 的区别是什么？
8. `sudo` 和 `su` 的区别是什么？
9. `usermod -aG group user` 中为什么要加 `-a`？
10. `umask 022` 通常会让新文件和新目录变成什么权限？
11. SUID、SGID、Sticky Bit 分别用什么字母显示？
12. 遇到 `Permission denied` 时，你会按什么顺序排查？

---

## 13. 阶段完成标准

当你能独立完成下面任务，就可以进入第四阶段：

- [ ] 使用 `id`、`groups` 查看用户身份和所属组。
- [ ] 读懂 `ls -l` 输出中的权限、所有者、所属组。
- [ ] 使用 `chmod` 设置 `644`、`600`、`755`、`700`。
- [ ] 使用符号方式添加或移除权限。
- [ ] 使用 `chown` 和 `chgrp` 修改所有者或所属组。
- [ ] 创建练习用户和练习组。
- [ ] 把用户加入指定组。
- [ ] 设置共享目录组权限和 SGID。
- [ ] 使用 `umask` 观察默认权限变化。
- [ ] 能排查一次真实或模拟的 `Permission denied`。

---

## 14. 建议写入学习笔记

记录下面内容：

```text
1. 我最常见的 4 个权限数字：644、600、755、700
2. 文件权限和目录权限的区别
3. sudo 使用前的检查清单
4. Permission denied 的排查流程
5. 本阶段我亲手创建的用户、组和共享目录实验结果
```

完成这些记录后，第三阶段就真正学完了。

