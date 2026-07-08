# 04：内存与 CPU 状态

本节目标：学会查看内存、swap、系统负载、CPU 使用情况，并初步判断系统是否资源紧张。

本节命令顺序：

```text
1. free -h
2. uptime
3. top
4. vmstat
5. vmstat 1 5
```

---

## 1. free：查看内存

执行：

```bash
free -h
```

常见列：

| 列 | 含义 |
| --- | --- |
| `total` | 总内存 |
| `used` | 已使用 |
| `free` | 完全空闲 |
| `buff/cache` | 缓存和缓冲 |
| `available` | 估计可用内存 |

重点看：

```text
available
```

Linux 会尽量使用空闲内存做缓存，所以 `free` 很低不一定代表内存不足。

---

## 2. swap 是什么

`free -h` 中还会看到 `Swap`。

swap 是磁盘上的交换空间。内存紧张时，系统可能把一部分内存数据换到 swap。

判断：

| 现象 | 可能含义 |
| --- | --- |
| swap 几乎不用 | 正常 |
| swap 长期大量使用 | 可能内存不足 |
| swap 使用高且系统很卡 | 需要进一步排查进程内存 |

---

## 3. uptime：查看负载

执行：

```bash
uptime
```

你会看到类似：

```text
load average: 0.15, 0.20, 0.18
```

三个数字分别表示：

```text
1 分钟、5 分钟、15 分钟平均负载
```

粗略理解：

```text
如果机器有 2 个 CPU 核心，负载长期高于 2，就可能比较忙。
```

查看 CPU 核心数：

```bash
nproc
```

---

## 4. top：实时查看 CPU 和内存

执行：

```bash
top
```

常用按键：

| 按键 | 作用 |
| --- | --- |
| `q` | 退出 |
| `P` | 按 CPU 排序 |
| `M` | 按内存排序 |
| `1` | 展开每个 CPU 核心 |

重点看：

- 哪个进程 CPU 高。
- 哪个进程内存高。
- load average 是否长期高。
- swap 是否大量使用。

---

## 5. vmstat：资源概览

执行：

```bash
vmstat
```

每 1 秒输出一次，共 5 次：

```bash
vmstat 1 5
```

常见列：

| 列 | 含义 |
| --- | --- |
| `r` | 等待运行的进程数 |
| `b` | 不可中断睡眠进程数，常见于 IO 等待 |
| `free` | 空闲内存 |
| `si` | swap in |
| `so` | swap out |
| `us` | 用户态 CPU |
| `sy` | 内核态 CPU |
| `id` | CPU 空闲 |
| `wa` | IO 等待 |

粗略判断：

| 现象 | 可能问题 |
| --- | --- |
| `us` 长期高 | 用户程序 CPU 忙 |
| `sy` 长期高 | 系统调用或内核开销高 |
| `wa` 长期高 | 磁盘 IO 可能慢 |
| `si/so` 长期不为 0 | swap 活跃，可能内存不足 |

---

## 6. 本节练习

执行：

```bash
free -h
uptime
nproc
top
vmstat
vmstat 1 5
```

回答：

1. 当前机器总内存是多少？
2. `available` 内存是多少？
3. 当前 load average 是多少？
4. CPU 核心数是多少？
5. `vmstat` 中 `wa` 是否明显偏高？

---

## 7. 本节小结

必须记住：

```bash
free -h
uptime
nproc
top
vmstat
vmstat 1 5
```

核心思路：

```text
内存看 available，CPU 看 top 和 load，IO 等待看 vmstat 的 wa。
```

完成后进入下一节：`05-io-monitoring.md`。

