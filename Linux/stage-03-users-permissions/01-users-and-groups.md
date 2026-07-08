# 01：用户与用户组基础

本节目标：理解当前用户是谁、用户属于哪些组，以及 Linux 为什么要区分用户和用户组。

本节命令顺序：

```text
1. whoami
2. id
3. groups
4. who
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-03
```

---

## 1. Linux 为什么需要用户和组

Linux 是多用户系统。即使你只有一台电脑，也可能同时存在很多用户：

- 你自己登录使用的普通用户。
- root 管理员用户。
- 系统服务用户，例如 `www-data`、`mysql`、`nginx`。

用户组用来把多个用户放到一起，然后统一分配权限。

例如：

```text
project 组可以写入 project 目录
dev 组可以读取代码目录
ops 组可以管理日志目录
```

---

## 2. whoami：查看当前用户

执行：

```bash
whoami
```

示例输出：

```text
study
```

这表示当前命令是由 `study` 用户执行的。

### 练习

```bash
whoami
pwd
```

记录当前用户名。

---

## 3. id：查看用户 ID 和组 ID

执行：

```bash
id
```

你可能看到：

```text
uid=1000(study) gid=1000(study) groups=1000(study),27(sudo)
```

含义：

| 字段 | 含义 |
| --- | --- |
| `uid` | 用户 ID |
| `gid` | 主用户组 ID |
| `groups` | 当前用户加入的所有组 |

Linux 内部更多依赖数字 ID，用户名和组名是给人看的。

### 查看指定用户

```bash
id root
```

root 通常是：

```text
uid=0(root) gid=0(root)
```

`uid=0` 表示超级管理员。

---

## 4. groups：查看当前用户所在组

执行：

```bash
groups
```

查看指定用户所在组：

```bash
groups root
```

如果你的用户在 `sudo` 组里，通常说明它可以使用 `sudo` 执行管理员命令。

---

## 5. who：查看当前登录用户

执行：

```bash
who
```

它会显示当前登录到系统的用户会话。

在 WSL 中输出可能较少，甚至没有明显结果，这是正常现象。真实服务器或虚拟机里会更常见。

---

## 6. 用户相关文件先认识一下

Linux 用户和组信息主要记录在这些文件中：

```text
/etc/passwd
/etc/group
/etc/shadow
```

查看用户列表的一部分：

```bash
head -n 5 /etc/passwd
```

查看组列表的一部分：

```bash
head -n 5 /etc/group
```

注意：

```text
/etc/shadow 存放密码相关信息，普通用户通常不能读取。
```

不要手动编辑这些文件，后面会使用专门命令管理用户和组。

---

## 7. 本节小结

必须记住：

```bash
whoami
id
id root
groups
who
head -n 5 /etc/passwd
head -n 5 /etc/group
```

核心概念：

- 用户决定“谁在操作”。
- 用户组决定“一类用户拥有什么权限”。
- root 是超级管理员，`uid=0`。
- 普通用户应该只在必要时使用管理员权限。

完成后进入下一节：`02-read-permissions.md`。

