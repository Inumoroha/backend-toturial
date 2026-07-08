# 04：所有者与所属组

本节目标：理解文件所有者、所属组，并学会使用 `chown` 和 `chgrp` 修改它们。

本节命令顺序：

```text
1. ls -l
2. chgrp
3. chown
4. chown user:group
5. chown -R
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-03
```

---

## 1. 看所有者和所属组

执行：

```bash
ls -l docs/public.txt
```

示例：

```text
-rw-r--r-- 1 study study 12 Jul  3 10:00 docs/public.txt
```

其中：

```text
第一个 study：所有者
第二个 study：所属组
```

权限判断顺序大致是：

```text
如果你是所有者，看用户权限。
如果你不是所有者但属于所属组，看组权限。
否则，看其他人权限。
```

---

## 2. chgrp：修改所属组

`chgrp` 用来修改文件所属组。

先查看当前用户所在组：

```bash
groups
```

如果要把文件所属组改成你自己的主组，可以先用：

```bash
id -gn
```

然后执行：

```bash
chgrp "$(id -gn)" docs/public.txt
ls -l docs/public.txt
```

如果要改成其他组，通常需要你属于那个组，或者使用 `sudo`。

---

## 3. chown：修改所有者

`chown` 用来修改文件所有者。

普通用户通常不能随便把文件改成别人所有，所以这个命令常需要 `sudo`。

查看当前文件：

```bash
ls -l docs/public.txt
```

把文件所有者改成当前用户：

```bash
sudo chown "$(whoami)" docs/public.txt
ls -l docs/public.txt
```

这个例子看起来变化不大，因为文件本来通常就是你的。

---

## 4. 同时修改所有者和所属组

格式：

```bash
sudo chown user:group file
```

把文件改成当前用户和当前主组：

```bash
sudo chown "$(whoami):$(id -gn)" docs/public.txt
ls -l docs/public.txt
```

只修改所属组也可以写成：

```bash
sudo chown :"$(id -gn)" docs/public.txt
```

---

## 5. 递归修改所有者

`-R` 会递归修改目录和里面所有内容。

只在练习目录中执行：

```bash
mkdir -p tmp/owner-demo/a
touch tmp/owner-demo/a/file.txt
sudo chown -R "$(whoami):$(id -gn)" tmp/owner-demo
ls -ld tmp/owner-demo tmp/owner-demo/a
ls -l tmp/owner-demo/a/file.txt
```

危险提醒：

```text
不要对系统目录乱用 chown -R。
不要执行 sudo chown -R user:user /。
```

---

## 6. chown 和 chmod 的区别

| 命令 | 修改什么 |
| --- | --- |
| `chmod` | 修改读、写、执行权限 |
| `chown` | 修改所有者 |
| `chgrp` | 修改所属组 |

示例：

```bash
chmod 600 private/secret.txt
sudo chown "$(whoami):$(id -gn)" private/secret.txt
chgrp "$(id -gn)" private/secret.txt
```

---

## 7. 本节小结

必须记住：

```bash
ls -l file
id -gn
chgrp group file
sudo chown user file
sudo chown user:group file
sudo chown -R user:group dir
```

核心概念：

```text
权限决定能做什么，所有者和所属组决定按哪一组权限判断。
```

完成后进入下一节：`05-sudo-su-users.md`。

