# 02：块设备与文件系统

本节目标：认识磁盘、分区、文件系统和挂载点，学会用 `lsblk`、`blkid`、`findmnt` 查看它们之间的关系。

本节命令顺序：

```text
1. lsblk
2. lsblk -f
3. lsblk -o
4. blkid
5. findmnt
6. findmnt /
```

---

## 1. 磁盘、分区、文件系统、挂载点

先理解四个概念：

| 概念 | 简单解释 |
| --- | --- |
| 磁盘 | 物理或虚拟存储设备，例如 `/dev/sda` |
| 分区 | 磁盘上划分出的区域，例如 `/dev/sda1` |
| 文件系统 | 管理文件存放方式的格式，例如 `ext4`、`xfs` |
| 挂载点 | 文件系统接入目录树的位置，例如 `/`、`/home` |

Linux 访问磁盘不是通过 `C:`、`D:`，而是把文件系统挂载到某个目录。

---

## 2. lsblk：查看块设备

执行：

```bash
lsblk
```

你可能看到：

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
```

重点看：

| 列 | 含义 |
| --- | --- |
| `NAME` | 设备名 |
| `SIZE` | 大小 |
| `TYPE` | 类型，disk 或 part |
| `MOUNTPOINTS` | 挂载点 |

---

## 3. lsblk -f：查看文件系统

执行：

```bash
lsblk -f
```

重点看：

| 列 | 含义 |
| --- | --- |
| `FSTYPE` | 文件系统类型 |
| `LABEL` | 标签 |
| `UUID` | 唯一标识 |
| `FSAVAIL` | 文件系统可用空间 |
| `FSUSE%` | 文件系统使用率 |
| `MOUNTPOINTS` | 挂载点 |

---

## 4. 自定义 lsblk 输出

执行：

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

这个输出更适合学习。

查看设备路径：

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

---

## 5. blkid：查看设备 UUID

执行：

```bash
sudo blkid
```

它会显示设备的文件系统类型和 UUID。

UUID 常用于 `/etc/fstab`，因为设备名可能变化，但 UUID 更稳定。

本阶段只要求看懂，不要求手动修改 `/etc/fstab`。

---

## 6. findmnt：查看挂载关系

查看所有挂载：

```bash
findmnt
```

查看根目录挂载：

```bash
findmnt /
```

查看当前目录所在挂载：

```bash
findmnt -T .
```

输出重点：

| 列 | 含义 |
| --- | --- |
| `TARGET` | 挂载点 |
| `SOURCE` | 来源设备或文件系统 |
| `FSTYPE` | 文件系统类型 |
| `OPTIONS` | 挂载选项 |

---

## 7. 本节练习

执行：

```bash
lsblk
lsblk -f
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo blkid
findmnt
findmnt /
findmnt -T .
```

回答：

1. 根目录 `/` 挂载在哪个设备或文件系统上？
2. 根目录的文件系统类型是什么？
3. 当前练习目录所在挂载点是什么？
4. `lsblk` 和 `df -h` 的视角有什么不同？

---

## 8. 本节小结

必须记住：

```bash
lsblk
lsblk -f
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo blkid
findmnt
findmnt /
findmnt -T .
```

核心概念：

```text
设备提供存储，文件系统管理文件，挂载点把文件系统接入目录树。
```

完成后进入下一节：`03-mount-umount.md`。

