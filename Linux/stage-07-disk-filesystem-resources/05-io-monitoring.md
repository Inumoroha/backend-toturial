# 05：IO 与磁盘性能观察

本节目标：学会使用 `iostat` 和 `vmstat` 观察磁盘 IO，理解 IO 等待和磁盘读写压力。

本节命令顺序：

```text
1. iostat
2. iostat -x
3. iostat -xz 1 5
4. vmstat 1 5
5. dd 写入测试文件
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-07
```

---

## 1. 安装 iostat

`iostat` 来自 `sysstat` 软件包。

安装：

```bash
sudo apt update
sudo apt install sysstat
```

查看：

```bash
iostat
```

---

## 2. iostat 基础输出

执行：

```bash
iostat
```

它通常显示：

- CPU 平均使用情况。
- 每个块设备的读写统计。

更详细：

```bash
iostat -x
```

常见列：

| 列 | 含义 |
| --- | --- |
| `r/s` | 每秒读请求 |
| `w/s` | 每秒写请求 |
| `rkB/s` | 每秒读取 KB |
| `wkB/s` | 每秒写入 KB |
| `%util` | 设备忙碌程度 |

不同版本输出列可能略有不同。

---

## 3. iostat -xz 1 5：连续观察

执行：

```bash
iostat -xz 1 5
```

含义：

```text
每 1 秒输出一次，共 5 次，显示扩展统计，隐藏全 0 设备。
```

如果 `%util` 长期很高，且系统响应慢，可能存在磁盘 IO 压力。

---

## 4. 用 vmstat 观察 IO 等待

执行：

```bash
vmstat 1 5
```

重点看：

| 列 | 含义 |
| --- | --- |
| `bi` | 从块设备读入 |
| `bo` | 写到块设备 |
| `wa` | IO 等待 CPU 百分比 |

`wa` 长期偏高时，说明 CPU 有不少时间在等 IO。

---

## 5. 制造一个小型写入实验

注意：下面命令只在练习目录写入一个 128M 文件。

执行：

```bash
cd ~/linux-practice/stage-07
dd if=/dev/zero of=tmp/io-test.bin bs=1M count=128 status=progress
sync
```

查看文件：

```bash
ls -lh tmp/io-test.bin
```

删除练习文件：

```bash
rm tmp/io-test.bin
```

如果你的磁盘空间很小，可以把 `count=128` 改成 `count=32`。

---

## 6. 一边写入一边观察

终端 A：

```bash
iostat -xz 1
```

终端 B：

```bash
cd ~/linux-practice/stage-07
dd if=/dev/zero of=tmp/io-test.bin bs=1M count=128 status=progress
sync
rm tmp/io-test.bin
```

观察写入时 `iostat` 的变化。

停止终端 A：

```text
Ctrl + C
```

---

## 7. 本节小结

必须记住：

```bash
sudo apt install sysstat
iostat
iostat -x
iostat -xz 1 5
vmstat 1 5
dd if=/dev/zero of=tmp/io-test.bin bs=1M count=128 status=progress
sync
```

核心思路：

```text
系统慢不一定是 CPU，可能是磁盘 IO 等待高。
```

完成后进入下一节：`06-lsof-open-files.md`。

