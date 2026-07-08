# 02：读懂 Linux 权限

本节目标：能读懂 `ls -l` 输出，理解文件权限、目录权限、所有者和所属组。

本节命令顺序：

```text
1. ls -l
2. ls -ld
3. stat
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-03
```

---

## 1. 用 ls -l 查看权限

执行：

```bash
ls -l
```

你可能看到：

```text
drwxr-xr-x 2 study study 4096 Jul  3 10:00 docs
-rw-r--r-- 1 study study   12 Jul  3 10:00 file.txt
```

重点看第一列：

```text
drwxr-xr-x
-rw-r--r--
```

它表示文件类型和权限。

---

## 2. 权限字符串怎么拆

以这个为例：

```text
-rw-r--r--
```

拆开：

```text
-    rw-   r--   r--
类型  用户  用户组  其他人
```

| 位置 | 含义 |
| --- | --- |
| 第 1 位 | 文件类型 |
| 第 2-4 位 | 所有者权限 |
| 第 5-7 位 | 所属组权限 |
| 第 8-10 位 | 其他人权限 |

常见文件类型：

| 符号 | 含义 |
| --- | --- |
| `-` | 普通文件 |
| `d` | 目录 |
| `l` | 符号链接 |

---

## 3. rwx 分别是什么意思

| 符号 | 英文 | 对文件的含义 | 对目录的含义 |
| --- | --- | --- | --- |
| `r` | read | 可以读取文件内容 | 可以列出目录内容 |
| `w` | write | 可以修改文件内容 | 可以在目录中创建、删除、重命名文件 |
| `x` | execute | 可以执行文件 | 可以进入目录 |

目录的 `x` 很重要。没有目录执行权限，即使有读权限，也不能正常进入目录。

---

## 4. 读懂几个常见权限

```text
-rw-r--r--
```

含义：

```text
普通文件；所有者可读写；组用户可读；其他人可读
```

```text
-rwxr-xr-x
```

含义：

```text
普通文件；所有者可读写执行；组用户可读执行；其他人可读执行
```

```text
drwxr-xr-x
```

含义：

```text
目录；所有者可读写进入；组用户可读进入；其他人可读进入
```

```text
drwx------
```

含义：

```text
目录；只有所有者可以读、写、进入
```

---

## 5. ls -ld：查看目录本身权限

如果执行：

```bash
ls -l docs
```

你看到的是 `docs` 目录里面的内容。

如果想看 `docs` 目录本身权限，需要：

```bash
ls -ld docs
```

### 练习

```bash
ls -l
ls -ld docs
ls -ld private
ls -l docs
ls -l private
```

---

## 6. stat：查看更详细的信息

执行：

```bash
stat docs/public.txt
```

重点看：

```text
Access: (0644/-rw-r--r--)
Uid:    用户 ID 和所有者
Gid:    组 ID 和所属组
```

查看目录：

```bash
stat docs
```

`stat` 会同时显示数字权限和符号权限。

---

## 7. 本节练习

执行下面命令，逐行解释输出：

```bash
cd ~/linux-practice/stage-03
ls -l
ls -ld docs scripts private
stat docs/public.txt
stat scripts/hello.sh
```

请回答：

1. `docs/public.txt` 的所有者是谁？
2. `docs/public.txt` 的所属组是谁？
3. `scripts/hello.sh` 当前有没有执行权限？
4. `private` 目录其他人是否可以进入？

---

## 8. 本节小结

必须记住：

```bash
ls -l
ls -ld dir
stat file
```

核心概念：

```text
rwx 权限分三组：所有者、所属组、其他人。
文件的 x 表示能执行，目录的 x 表示能进入。
```

完成后进入下一节：`03-chmod.md`。

