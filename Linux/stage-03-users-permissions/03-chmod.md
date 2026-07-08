# 03：chmod 修改权限

本节目标：学会使用数字方式和符号方式修改文件、目录权限。

本节命令顺序：

```text
1. chmod u+x
2. chmod g+w
3. chmod o-r
4. chmod 644
5. chmod 600
6. chmod 755
7. chmod 700
8. chmod -R
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-03
```

---

## 1. chmod 是什么

`chmod` 用来修改权限。

权限有两种常见写法：

```text
符号方式：chmod u+x file
数字方式：chmod 755 file
```

符号方式适合小范围调整，数字方式适合设置完整权限。

---

## 2. 符号方式

符号方式由三部分组成：

```text
谁   操作   权限
u    +      x
```

### 谁

| 符号 | 含义 |
| --- | --- |
| `u` | user，所有者 |
| `g` | group，所属组 |
| `o` | others，其他人 |
| `a` | all，所有人 |

### 操作

| 符号 | 含义 |
| --- | --- |
| `+` | 添加权限 |
| `-` | 移除权限 |
| `=` | 设置为指定权限 |

### 权限

| 符号 | 含义 |
| --- | --- |
| `r` | 读 |
| `w` | 写 |
| `x` | 执行或进入 |

---

## 3. 给脚本添加执行权限

先查看权限：

```bash
ls -l scripts/hello.sh
```

尝试执行：

```bash
./scripts/hello.sh
```

如果没有执行权限，你会看到：

```text
Permission denied
```

给所有者添加执行权限：

```bash
chmod u+x scripts/hello.sh
```

再次查看：

```bash
ls -l scripts/hello.sh
```

执行：

```bash
./scripts/hello.sh
```

---

## 4. 移除和设置权限

给组用户添加写权限：

```bash
chmod g+w docs/public.txt
ls -l docs/public.txt
```

移除其他人的读权限：

```bash
chmod o-r docs/public.txt
ls -l docs/public.txt
```

设置所有者可读写，组和其他人无权限：

```bash
chmod u=rw,g=,o= docs/public.txt
ls -l docs/public.txt
```

恢复为常见普通文件权限：

```bash
chmod 644 docs/public.txt
ls -l docs/public.txt
```

---

## 5. 数字权限

数字权限由三位数字组成：

```text
所有者  所属组  其他人
7       5       5
```

权限数字：

| 权限 | 数字 |
| --- | --- |
| `r` | 4 |
| `w` | 2 |
| `x` | 1 |

组合方式：

| 数字 | 权限 | 含义 |
| --- | --- | --- |
| `7` | `rwx` | 读、写、执行 |
| `6` | `rw-` | 读、写 |
| `5` | `r-x` | 读、执行 |
| `4` | `r--` | 只读 |
| `0` | `---` | 无权限 |

---

## 6. 常见权限组合

| 权限 | 符号形式 | 常见用途 |
| --- | --- | --- |
| `644` | `rw-r--r--` | 普通文本文件 |
| `600` | `rw-------` | 私密文件 |
| `755` | `rwxr-xr-x` | 脚本或普通目录 |
| `700` | `rwx------` | 私密目录或私密脚本 |

### 练习

普通文件：

```bash
chmod 644 docs/public.txt
ls -l docs/public.txt
```

私密文件：

```bash
chmod 600 private/secret.txt
ls -l private/secret.txt
```

可执行脚本：

```bash
chmod 755 scripts/hello.sh
ls -l scripts/hello.sh
```

私密目录：

```bash
chmod 700 private
ls -ld private
```

---

## 7. chmod -R：递归修改权限

`-R` 会递归修改目录和目录里的所有内容。

示例：

```bash
chmod -R 700 private
```

危险提醒：

```text
不要对系统目录乱用 chmod -R。
不要执行 chmod -R 777 / 这类命令。
```

只在自己的练习目录中使用：

```bash
mkdir -p tmp/chmod-demo/a
touch tmp/chmod-demo/a/file.txt
chmod -R 700 tmp/chmod-demo
ls -ld tmp/chmod-demo tmp/chmod-demo/a
ls -l tmp/chmod-demo/a/file.txt
```

---

## 8. 本节小结

必须记住：

```bash
chmod u+x script.sh
chmod g+w file.txt
chmod o-r file.txt
chmod 644 file.txt
chmod 600 secret.txt
chmod 755 script.sh
chmod 700 private-dir
chmod -R 700 dir
```

最常用权限：

```text
普通文件：644
私密文件：600
脚本或目录：755
私密目录：700
```

完成后进入下一节：`04-owner-and-group.md`。

