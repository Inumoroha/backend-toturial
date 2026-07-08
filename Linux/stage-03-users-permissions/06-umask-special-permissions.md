# 06：umask 与特殊权限

本节目标：理解新文件默认权限从哪里来，并认识 SUID、SGID、Sticky Bit 三种特殊权限。

本节命令顺序：

```text
1. umask
2. touch
3. mkdir
4. chmod g+s
5. chmod +t
6. find -perm
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-03
```

---

## 1. umask 是什么

`umask` 决定新建文件和目录默认会去掉哪些权限。

查看当前 `umask`：

```bash
umask
```

常见输出：

```text
0022
```

一般情况下：

```text
新文件基础权限：666
新目录基础权限：777
```

如果 `umask` 是 `022`：

```text
新文件默认权限：644
新目录默认权限：755
```

---

## 2. 观察默认权限

执行：

```bash
cd ~/linux-practice/stage-03
mkdir -p tmp/umask-demo
cd tmp/umask-demo
umask
touch file-default.txt
mkdir dir-default
ls -l
ls -ld dir-default
```

观察：

- 新文件是不是通常没有执行权限？
- 新目录是不是通常有执行权限？

原因：

```text
文件默认不应该可执行，目录需要 x 权限才能进入。
```

---

## 3. 临时修改 umask

只影响当前 Shell：

```bash
umask 077
touch file-private.txt
mkdir dir-private
ls -l
ls -ld dir-private
```

你会看到更私密的权限，例如：

```text
-rw-------
drwx------
```

恢复常见设置：

```bash
umask 022
```

再次创建：

```bash
touch file-normal.txt
mkdir dir-normal
ls -l
ls -ld dir-normal
```

---

## 4. SUID 基础认识

SUID 让用户执行某个程序时，临时拥有文件所有者的权限。

查看系统中常见 SUID 程序：

```bash
ls -l /usr/bin/passwd
```

你可能看到：

```text
-rwsr-xr-x
```

其中 `s` 表示 SUID。

普通用户能修改自己的密码，是因为 `passwd` 程序需要临时访问受保护的密码相关文件。

本阶段只要求认识 SUID，不建议自己随意设置 SUID。

---

## 5. SGID 基础认识

SGID 用在目录上时，新建文件会继承该目录的所属组。

创建练习目录：

```bash
cd ~/linux-practice/stage-03
mkdir -p shared/sgid-demo
chmod g+s shared/sgid-demo
ls -ld shared/sgid-demo
```

你可能看到：

```text
drwxr-sr-x
```

组执行位上的 `s` 表示 SGID。

在团队共享目录中，SGID 很常见。

---

## 6. Sticky Bit 基础认识

Sticky Bit 常用于公共可写目录。它允许大家写入，但只有文件所有者、目录所有者或 root 能删除文件。

系统临时目录通常有 Sticky Bit：

```bash
ls -ld /tmp
```

你可能看到：

```text
drwxrwxrwt
```

最后的 `t` 表示 Sticky Bit。

给练习目录设置 Sticky Bit：

```bash
cd ~/linux-practice/stage-03
mkdir -p shared/sticky-demo
chmod 1777 shared/sticky-demo
ls -ld shared/sticky-demo
```

也可以写成：

```bash
chmod +t shared/sticky-demo
```

---

## 7. find -perm：查找特殊权限文件

查找 SUID 文件：

```bash
find /usr/bin -perm -4000 -type f 2>/dev/null | head
```

查找 SGID 文件：

```bash
find /usr/bin -perm -2000 -type f 2>/dev/null | head
```

查找 Sticky Bit 目录：

```bash
find /tmp -perm -1000 -type d 2>/dev/null | head
```

这里的 `2>/dev/null` 是把无权限访问的错误信息丢弃。

---

## 8. 本节小结

必须记住：

```bash
umask
umask 077
umask 022
ls -l /usr/bin/passwd
chmod g+s shared-dir
chmod +t public-dir
find /usr/bin -perm -4000 -type f 2>/dev/null
```

本阶段对特殊权限的要求：

```text
知道它们是什么，能看懂 s 和 t，不需要大量手动设置。
```

完成后进入下一节：`07-stage-practice.md`。

