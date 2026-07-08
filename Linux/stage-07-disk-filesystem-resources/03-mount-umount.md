# 03：mount 与 umount 挂载实验

本节目标：理解挂载和卸载，学会查看挂载点，并安全完成一次 `tmpfs` 临时文件系统挂载实验。

本节命令顺序：

```text
1. mount
2. findmnt
3. sudo mount -t tmpfs
4. df -h
5. sudo umount
6. findmnt -T
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-07
```

---

## 1. 挂载是什么

挂载就是把一个文件系统接入 Linux 目录树。

例如：

```text
设备 /dev/sda1 -> 挂载到 /
tmpfs -> 挂载到 /run
```

挂载后，你访问挂载点目录，其实是在访问这个文件系统的内容。

---

## 2. 查看当前挂载

查看所有挂载：

```bash
mount
```

输出很多，更推荐：

```bash
findmnt
```

查看当前目录所在挂载：

```bash
findmnt -T .
```

---

## 3. tmpfs 是什么

`tmpfs` 是一种基于内存的临时文件系统。

特点：

- 不会格式化磁盘。
- 数据通常重启后消失。
- 适合做安全的挂载练习。

本节只用 `tmpfs` 练习，不做真实磁盘分区和格式化。

---

## 4. 挂载 tmpfs

创建挂载点：

```bash
mkdir -p mnt/tmpfs-demo
```

挂载 64M 的 tmpfs：

```bash
sudo mount -t tmpfs -o size=64M tmpfs mnt/tmpfs-demo
```

查看：

```bash
findmnt -T mnt/tmpfs-demo
df -h mnt/tmpfs-demo
```

写入文件：

```bash
echo "hello tmpfs" > mnt/tmpfs-demo/hello.txt
cat mnt/tmpfs-demo/hello.txt
```

---

## 5. 卸载 tmpfs

卸载前，确保当前终端不在挂载点里面。

先回到练习目录：

```bash
cd ~/linux-practice/stage-07
```

卸载：

```bash
sudo umount mnt/tmpfs-demo
```

确认：

```bash
findmnt -T mnt/tmpfs-demo
df -h mnt/tmpfs-demo
ls -lah mnt/tmpfs-demo
```

卸载后，刚才写在 tmpfs 中的文件通常就看不到了。

---

## 6. umount 失败的常见原因

如果看到：

```text
target is busy
```

常见原因：

- 某个终端当前目录在挂载点中。
- 某个进程正在使用挂载点里的文件。

排查：

```bash
lsof +D mnt/tmpfs-demo
```

或者：

```bash
fuser -vm mnt/tmpfs-demo
```

新手阶段知道原因即可，不要急着强制卸载。

---

## 7. /etc/fstab 先认识

系统启动时自动挂载的配置通常在：

```bash
cat /etc/fstab
```

本阶段只需要看懂大概结构，不建议新手随意修改。

常见字段：

```text
设备或 UUID   挂载点   文件系统类型   选项   dump   fsck
```

---

## 8. 本节练习

执行：

```bash
cd ~/linux-practice/stage-07
mkdir -p mnt/tmpfs-demo
sudo mount -t tmpfs -o size=64M tmpfs mnt/tmpfs-demo
findmnt -T mnt/tmpfs-demo
df -h mnt/tmpfs-demo
echo "tmpfs test" > mnt/tmpfs-demo/test.txt
cat mnt/tmpfs-demo/test.txt
cd ~/linux-practice/stage-07
sudo umount mnt/tmpfs-demo
findmnt -T mnt/tmpfs-demo
```

---

## 9. 本节小结

必须记住：

```bash
mount
findmnt
findmnt -T .
sudo mount -t tmpfs -o size=64M tmpfs mnt/tmpfs-demo
df -h mnt/tmpfs-demo
sudo umount mnt/tmpfs-demo
cat /etc/fstab
```

完成后进入下一节：`04-memory-cpu.md`。

