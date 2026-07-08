# 第七阶段：磁盘、文件系统与系统资源教程

本阶段目标：能查看磁盘空间、定位大文件、理解块设备和挂载点，能查看内存、CPU、IO 状态，并能排查磁盘空间不足、文件被占用、资源异常等常见问题。

建议学习时间：1 周。  
建议学习方式：按文件顺序学习，每节先看“本节命令顺序”，再跟着练习。  
练习目录：`~/linux-practice/stage-07`

---

## 学习文件顺序

| 顺序 | 文件 | 主题 | 重点命令 |
| --- | --- | --- | --- |
| 1 | `01-df-du-disk-usage.md` | 磁盘空间与目录占用 | `df`、`du`、`sort`、`head` |
| 2 | `02-lsblk-filesystems.md` | 块设备与文件系统 | `lsblk`、`blkid`、`findmnt` |
| 3 | `03-mount-umount.md` | 挂载与卸载 | `mount`、`umount`、`findmnt` |
| 4 | `04-memory-cpu.md` | 内存与 CPU | `free`、`uptime`、`top`、`vmstat` |
| 5 | `05-io-monitoring.md` | IO 与磁盘性能观察 | `iostat`、`vmstat`、`dd` |
| 6 | `06-lsof-open-files.md` | 打开文件与占用排查 | `lsof` |
| 7 | `07-ncdu-cleanup.md` | 交互式空间分析与清理 | `ncdu`、安全清理思路 |
| 8 | `08-stage-practice.md` | 第七阶段综合练习 | 磁盘、内存、IO、挂载、lsof 综合排查 |

---

## 第七阶段命令学习总顺序

建议按下面顺序学习：

```text
1. df -h
2. df -i
3. du -sh
4. du -ah
5. sort -rh
6. lsblk
7. lsblk -f
8. blkid
9. findmnt
10. mount
11. umount
12. free -h
13. uptime
14. top
15. vmstat
16. iostat
17. lsof
18. ncdu
```

---

## 开始前准备

创建第七阶段练习目录：

```bash
mkdir -p ~/linux-practice/stage-07/{data,logs,tmp,mnt,output}
cd ~/linux-practice/stage-07
pwd
```

创建一些练习文件：

```bash
echo "hello disk" > data/small.txt
seq 1 10000 > data/numbers.txt
truncate -s 20M data/big-20m.bin
truncate -s 5M logs/app.log
```

查看结构：

```bash
du -sh .
find . -maxdepth 2 -type f -exec ls -lh {} \;
```

---

## 本阶段安全提醒

本阶段涉及磁盘和挂载，请遵守：

- 不要对真实磁盘执行 `mkfs`、`fdisk`、`parted` 等命令，除非你明确知道后果。
- 不要随便卸载系统正在使用的挂载点。
- 不要对 `/`、`/home`、`/var` 这类目录做盲目删除。
- 清理文件前先用 `du`、`lsof`、`findmnt` 确认。
- 本教程的挂载练习使用 `tmpfs` 临时文件系统，不会格式化磁盘。

---

## 本阶段排查思路

遇到“磁盘满了”可以按这个顺序：

```text
1. df -h 看哪个挂载点满了
2. df -i 看 inode 是否满了
3. du -sh /* 或目标目录，看哪个目录大
4. du -ah 目标目录 | sort -rh | head 定位大文件
5. lsof 检查是否有已删除但仍被进程占用的文件
6. 确认文件用途后再清理
```

遇到“系统变慢”可以按这个顺序：

```text
1. uptime 看负载
2. free -h 看内存
3. top 看 CPU 和进程
4. vmstat 看 CPU、内存、IO 概况
5. iostat 看磁盘 IO
```

---

## 本阶段验收标准

完成本阶段后，你应该能做到：

- 能用 `df -h` 查看磁盘空间。
- 能用 `df -i` 查看 inode 使用情况。
- 能用 `du` 定位目录和文件占用。
- 能解释 `df` 和 `du` 的区别。
- 能用 `lsblk` 查看磁盘、分区、挂载点。
- 能用 `findmnt` 查看挂载关系。
- 能安全完成一次 `tmpfs` 挂载和卸载实验。
- 能用 `free`、`uptime`、`top`、`vmstat` 查看资源状态。
- 能用 `iostat` 初步观察磁盘 IO。
- 能用 `lsof` 排查文件或端口被占用。
- 能用 `ncdu` 交互式分析目录空间。

