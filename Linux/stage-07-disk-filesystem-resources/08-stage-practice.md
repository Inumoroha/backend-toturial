# 08：第七阶段综合练习

本节目标：完成一次磁盘空间、挂载、内存、CPU、IO、打开文件和清理流程的综合实验。

请在 WSL、虚拟机或测试环境中练习。本练习不会要求你格式化磁盘，也不会修改真实分区表。

---

## 1. 本练习会用到的命令

按出现顺序：

```text
1. mkdir
2. truncate
3. df -h
4. df -i
5. du -sh
6. du -ah
7. sort -rh
8. head
9. lsblk
10. findmnt
11. mount
12. umount
13. free -h
14. uptime
15. top
16. vmstat
17. iostat
18. lsof
19. ncdu
20. rm -i
```

---

## 2. 初始化实验目录

```bash
mkdir -p ~/linux-practice/stage-07/final-lab/{data,logs,tmp,mnt,output}
cd ~/linux-practice/stage-07/final-lab
pwd
```

创建不同大小的文件：

```bash
echo "small file" > data/small.txt
seq 1 50000 > data/numbers.txt
truncate -s 30M data/big-30m.bin
truncate -s 10M logs/app.log
truncate -s 5M tmp/cache.bin
```

查看：

```bash
find . -maxdepth 2 -type f -exec ls -lh {} \;
```

---

## 3. 磁盘空间检查

查看当前文件系统空间：

```bash
df -h .
```

查看 inode：

```bash
df -i .
```

查看实验目录总大小：

```bash
du -sh .
```

查看一级目录大小：

```bash
du -h --max-depth=1 .
```

找出最大的 10 个文件或目录：

```bash
du -ah . | sort -rh | head -n 10
```

写入报告：

```bash
echo "# Stage 07 Report" > output/report.md
echo "## Disk" >> output/report.md
df -h . >> output/report.md
echo "## Top Usage" >> output/report.md
du -ah . | sort -rh | head -n 10 >> output/report.md
```

---

## 4. 块设备与挂载关系检查

查看块设备：

```bash
lsblk
lsblk -f
```

查看当前目录所在挂载点：

```bash
findmnt -T .
```

查看根目录挂载：

```bash
findmnt /
```

追加报告：

```bash
echo "## Mount For Current Directory" >> output/report.md
findmnt -T . >> output/report.md
```

---

## 5. tmpfs 挂载实验

创建挂载点：

```bash
mkdir -p mnt/tmpfs-lab
```

挂载 64M tmpfs：

```bash
sudo mount -t tmpfs -o size=64M tmpfs mnt/tmpfs-lab
```

检查：

```bash
findmnt -T mnt/tmpfs-lab
df -h mnt/tmpfs-lab
```

写入文件：

```bash
echo "hello tmpfs lab" > mnt/tmpfs-lab/hello.txt
cat mnt/tmpfs-lab/hello.txt
```

卸载：

```bash
cd ~/linux-practice/stage-07/final-lab
sudo umount mnt/tmpfs-lab
findmnt -T mnt/tmpfs-lab
```

---

## 6. 内存和 CPU 检查

查看内存：

```bash
free -h
```

查看负载：

```bash
uptime
nproc
```

查看实时进程：

```bash
top
```

在 `top` 中按 `q` 退出。

查看资源概览：

```bash
vmstat 1 5
```

追加报告：

```bash
echo "## Memory" >> output/report.md
free -h >> output/report.md
echo "## Load" >> output/report.md
uptime >> output/report.md
```

---

## 7. IO 观察实验

如果没有 `iostat`：

```bash
sudo apt install sysstat
```

查看 IO：

```bash
iostat -xz 1 3
```

写入一个测试文件：

```bash
dd if=/dev/zero of=tmp/io-test.bin bs=1M count=64 status=progress
sync
ls -lh tmp/io-test.bin
```

再次观察：

```bash
iostat -xz 1 3
```

删除测试文件：

```bash
rm tmp/io-test.bin
```

---

## 8. lsof 占用排查实验

创建日志：

```bash
echo "lsof lab" > logs/lsof-demo.log
```

终端 A 执行：

```bash
tail -f logs/lsof-demo.log
```

终端 B 进入目录：

```bash
cd ~/linux-practice/stage-07/final-lab
```

查看文件占用：

```bash
lsof logs/lsof-demo.log
```

查看进程：

```bash
pgrep -af "tail -f logs/lsof-demo.log"
```

回到终端 A，按：

```text
Ctrl + C
```

---

## 9. ncdu 与安全清理

安装：

```bash
sudo apt install ncdu
```

扫描当前目录：

```bash
ncdu .
```

命令行复核：

```bash
du -ah . | sort -rh | head -n 10
```

创建可删除文件：

```bash
truncate -s 12M tmp/delete-me.bin
ls -lh tmp/delete-me.bin
```

删除前确认：

```bash
du -sh tmp/delete-me.bin
rm -i tmp/delete-me.bin
```

删除后验证：

```bash
du -sh .
df -h .
```

---

## 10. 磁盘空间不足排查模板

以后遇到磁盘空间不足，可以按顺序执行：

```bash
df -h
df -i
du -h --max-depth=1 /
du -ah /path/to/big-dir | sort -rh | head -n 20
sudo lsof | grep deleted
```

如果是自己的项目目录：

```bash
cd /path/to/project
du -h --max-depth=1 .
du -ah . | sort -rh | head -n 20
ncdu .
```

---

## 11. 系统资源异常排查模板

系统卡顿时：

```bash
uptime
free -h
top
vmstat 1 5
iostat -xz 1 5
```

判断方向：

| 现象 | 方向 |
| --- | --- |
| load 高，CPU 高 | 查 CPU 忙的进程 |
| available 内存低，swap 活跃 | 查内存占用 |
| vmstat 的 wa 高 | 查 IO |
| df 空间满 | 查大文件和已删除占用 |

---

## 12. 查看报告

```bash
cat output/report.md
```

你也可以继续追加：

```bash
echo "## Final Directory Size" >> output/report.md
du -sh . >> output/report.md
```

---

## 13. 阶段验收题

请你不看答案，回答下面问题：

1. `df -h` 和 `du -sh` 有什么区别？
2. `df -i` 是查看什么？
3. `du -ah . | sort -rh | head` 是做什么的？
4. `lsblk` 和 `findmnt` 的区别是什么？
5. 挂载点是什么？
6. `tmpfs` 有什么特点？
7. 卸载时提示 `target is busy` 可能是什么原因？
8. `free -h` 中应该重点看 `free` 还是 `available`？
9. `uptime` 的 load average 三个数字分别表示什么？
10. `vmstat` 中 `wa` 偏高可能说明什么？
11. `iostat -xz 1 5` 是做什么的？
12. 文件删除后空间不释放，应该用什么命令排查？
13. 清理大文件前应该确认哪些信息？

---

## 14. 阶段完成标准

当你能独立完成下面任务，就可以进入第八阶段：

- [ ] 使用 `df -h` 查看磁盘空间。
- [ ] 使用 `df -i` 查看 inode。
- [ ] 使用 `du` 找出大目录和大文件。
- [ ] 使用 `lsblk` 查看块设备。
- [ ] 使用 `findmnt` 查看挂载点。
- [ ] 完成一次 `tmpfs` 挂载和卸载。
- [ ] 使用 `free -h` 查看内存。
- [ ] 使用 `uptime` 和 `top` 查看负载和进程。
- [ ] 使用 `vmstat` 查看资源概况。
- [ ] 使用 `iostat` 观察 IO。
- [ ] 使用 `lsof` 查看文件或端口占用。
- [ ] 使用 `ncdu` 分析目录空间。
- [ ] 写出一套磁盘空间不足排查流程。

---

## 15. 建议写入学习笔记

记录下面内容：

```text
1. df 和 du 的区别
2. 我的磁盘空间排查流程
3. lsblk 输出中每一列的含义
4. mount 和 umount 的使用注意事项
5. free、uptime、top、vmstat、iostat 分别看什么
6. lsof 的三个常见用途
7. 一次完整的空间清理实验过程
```

完成这些记录后，第七阶段就真正学完了。

