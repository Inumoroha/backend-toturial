# 04：命令帮助系统

本节目标：遇到不熟悉的命令时，知道如何自己查用法。

本节命令顺序：

```text
1. command --help
2. man
3. type
4. which
5. whereis
```

---

## 1. command --help：快速帮助

大多数命令都支持 `--help`。

示例：

```bash
ls --help
```

查看 `mkdir` 的帮助：

```bash
mkdir --help
```

查看 `cp` 的帮助：

```bash
cp --help
```

适合用 `--help` 的场景：

- 忘了某个参数怎么写。
- 想快速看常用选项。
- 不想打开完整手册。

### 练习

执行：

```bash
ls --help
mkdir --help
cp --help
```

找出这些参数的含义：

```text
ls -l
ls -a
mkdir -p
cp -r
```

---

## 2. man：完整手册

`man` 是 manual 的缩写，用来查看命令手册。

查看 `ls` 手册：

```bash
man ls
```

查看 `cp` 手册：

```bash
man cp
```

进入 `man` 后常用按键：

| 按键 | 作用 |
| --- | --- |
| `空格` | 向下翻页 |
| `b` | 向上翻页 |
| `/关键词` | 搜索关键词 |
| `n` | 跳到下一个搜索结果 |
| `q` | 退出 |

### man 页面常见结构

| 区域 | 含义 |
| --- | --- |
| `NAME` | 命令名称和一句话说明 |
| `SYNOPSIS` | 命令格式 |
| `DESCRIPTION` | 详细说明 |
| `OPTIONS` | 参数说明 |
| `EXAMPLES` | 示例，有些手册才有 |

### 练习

执行：

```bash
man ls
```

在手册里搜索：

```text
/-a
```

然后按 `n` 查找下一个匹配项，最后按 `q` 退出。

---

## 3. type：判断命令来源

有些命令是 Shell 内置的，有些是系统里的可执行程序。

查看 `cd`：

```bash
type cd
```

查看 `ls`：

```bash
type ls
```

查看 `pwd`：

```bash
type pwd
```

你可能看到：

```text
cd is a shell builtin
ls is aliased to `ls --color=auto`
pwd is a shell builtin
```

这说明：

- `cd` 是 Shell 内置命令。
- `ls` 可能是一个别名。
- `pwd` 可能是 Shell 内置命令。

### 练习

执行：

```bash
type cd
type ls
type cat
type mkdir
```

观察哪些是内置命令，哪些是外部程序。

---

## 4. which：查找命令路径

`which` 用来查看某个外部命令的位置。

执行：

```bash
which ls
which cat
which mkdir
```

你可能看到：

```text
/usr/bin/ls
/usr/bin/cat
/usr/bin/mkdir
```

如果一个命令是 Shell 内置命令，`which` 不一定能给出你想要的结果，所以需要结合 `type` 使用。

### 练习

执行：

```bash
type cd
which cd
type ls
which ls
```

比较输出差异。

---

## 5. whereis：查找程序、源码和手册位置

`whereis` 可以查找命令程序、源码、man 手册位置。

执行：

```bash
whereis ls
whereis cat
whereis mkdir
```

你可能看到：

```text
ls: /usr/bin/ls /usr/share/man/man1/ls.1.gz
```

这表示 `ls` 程序和它的手册文件位置。

---

## 6. 查命令的推荐顺序

遇到陌生命令时，建议按这个顺序查：

```text
1. command --help
2. man command
3. type command
4. which command
5. whereis command
```

示例：查询 `tail`

```bash
tail --help
man tail
type tail
which tail
whereis tail
```

---

## 7. 本节小结

本节必须记住：

```bash
ls --help
man ls
type ls
which ls
whereis ls
```

真正重要的不是记住每个参数，而是遇到问题时知道怎么查。

完成后进入下一节：`05-stage-practice.md`。

