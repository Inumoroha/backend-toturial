# 01：df 与 du 查看磁盘空间

本节目标：学会查看文件系统剩余空间，统计目录占用，并定位大文件。

本节命令顺序：

```text
1. df -h
2. df -i
3. du -sh
4. du -h --max-depth=1
5. du -ah
6. sort -rh
7. head
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-07
```

---

## 1. df：查看文件系统空间

执行：

```bash
df
```

更常用的人类可读格式：

```bash
df -h
```

重点看：

| 列 | 含义 |
| --- | --- |
| `Filesystem` | 文件系统或设备 |
| `Size` | 总大小 |
| `Used` | 已使用 |
| `Avail` | 可用 |
| `Use%` | 使用率 |
| `Mounted on` | 挂载点 |

查看当前目录所在文件系统：

```bash
df -h .
```

---

## 2. df -i：查看 inode

磁盘空间没满，但仍然创建不了文件，可能是 inode 用完了。

查看 inode：

```bash
df -i
```

重点看：

| 列 | 含义 |
| --- | --- |
| `Inodes` | inode 总数 |
| `IUsed` | 已用 inode |
| `IFree` | 剩余 inode |
| `IUse%` | inode 使用率 |

inode 可以粗略理解为“文件数量额度”。大量小文件可能先耗尽 inode。

---

## 3. du：查看目录占用

查看当前目录总大小：

```bash
du -sh .
```

查看子目录大小：

```bash
du -sh *
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-s` | 汇总 |
| `-h` | 人类可读 |

---

## 4. 查看每层目录占用

查看当前目录下一层占用：

```bash
du -h --max-depth=1 .
```

查看家目录下一层：

```bash
du -h --max-depth=1 ~
```

如果权限不足，可能会看到 `Permission denied`。查看系统目录时可以使用 `sudo`，但先尽量在自己的目录练习。

---

## 5. 找出最大的文件或目录

查看当前目录下所有文件和目录大小：

```bash
du -ah .
```

按大小倒序：

```bash
du -ah . | sort -rh
```

只看前 10 个：

```bash
du -ah . | sort -rh | head -n 10
```

这是排查磁盘占用最常用的组合之一。

---

## 6. df 和 du 的区别

| 命令 | 看什么 |
| --- | --- |
| `df` | 文件系统整体使用情况 |
| `du` | 文件和目录占用空间 |

简单记法：

```text
df 看分区还剩多少。
du 看目录里谁占得多。
```

有时候 `df` 显示空间已用很多，但 `du` 找不到对应文件，可能是文件被删除后仍被进程占用。后面会用 `lsof` 排查。

---

## 7. 本节练习

执行：

```bash
cd ~/linux-practice/stage-07
df -h .
df -i .
du -sh .
du -sh *
du -h --max-depth=1 .
du -ah . | sort -rh | head -n 10
```

回答：

1. 当前目录所在文件系统总大小是多少？
2. 当前目录总共占用多少？
3. `data` 和 `logs` 哪个目录更大？
4. 最大的 3 个文件分别是什么？

---

## 8. 本节小结

必须记住：

```bash
df -h
df -h .
df -i
du -sh .
du -sh *
du -h --max-depth=1 .
du -ah . | sort -rh | head -n 10
```

完成后进入下一节：`02-lsblk-filesystems.md`。

