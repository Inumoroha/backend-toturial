# 4. 文件描述符、临时端口、TIME_WAIT、CLOSE_WAIT

本节目标：理解连接资源限制，能排查 fd 耗尽、临时端口耗尽、TIME_WAIT 和 CLOSE_WAIT。

---

## 一、文件描述符

Linux 中 socket 也是文件描述符。

查看限制：

```bash
ulimit -n
cat /proc/进程ID/limits
```

查看进程 fd 数：

```bash
ls /proc/进程ID/fd | wc -l
```

fd 耗尽可能出现：

```text
too many open files
```

---

## 二、临时端口

客户端发起连接时使用本地临时端口。

查看范围：

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

大量短连接可能导致临时端口压力。

---

## 三、TIME_WAIT

查看：

```bash
ss -ant state time-wait | wc -l
```

常见原因：

- 短连接多。
- 没有复用连接池。
- 主动关闭集中在本机。

---

## 四、CLOSE_WAIT

查看：

```bash
ss -ant state close-wait | wc -l
```

常见原因：

- 程序没有关闭连接。
- HTTP Client 没有关闭 `resp.Body`。
- TCP 读写循环错误处理不完整。

---

## 五、Go 排查重点

检查代码：

```go
defer resp.Body.Close()
defer conn.Close()
```

检查 goroutine：

```bash
curl http://127.0.0.1:6060/debug/pprof/goroutine?debug=1
```

---

## 补充排查：fd 是否正在持续增长

只看一次 fd 数量意义有限，要观察趋势：

```bash
pid=进程ID
while true; do
  date
  ls /proc/$pid/fd | wc -l
  sleep 2
done
```

如果 fd 持续增长且不回落，常见原因是：

```text
HTTP response body 没关闭。
TCP conn 没关闭。
文件没关闭。
goroutine 卡住导致 defer 没执行。
```

再看 fd 指向什么：

```bash
ls -l /proc/$pid/fd | head
lsof -p $pid | head -50
```

如果大量是：

```text
TCP ...
socket:[...]
```

优先查网络连接泄漏。

---

## 补充排查：临时端口是否耗尽

查看端口范围：

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

统计到某个上游的连接：

```bash
ss -antp | grep '目标IP:目标端口' | wc -l
```

如果大量短连接处于 `TIME_WAIT`，并且请求方不断报：

```text
cannot assign requested address
```

可能是临时端口压力过大。

常见修复方向：

```text
复用 HTTP Client。
开启连接池。
减少无意义短连接。
检查上游是否主动关闭导致无法复用。
合理调整系统参数。
```

系统参数可以缓解，但代码里的连接复用才是根本。

---

## 补充复盘：资源类故障要看趋势

资源类问题很少只靠一个瞬间截图定位。建议同时记录：

```text
fd 数量随时间变化。
goroutine 数量随时间变化。
ESTABLISHED 数量。
CLOSE_WAIT 数量。
TIME_WAIT 数量。
错误日志出现时间。
发布或流量变化时间。
```

如果这些指标在发布后持续爬升，就要优先怀疑新代码引入泄漏。

如果这些指标只在流量峰值上升、峰值后回落，更可能是容量或连接池配置问题。

这两种问题的处理方式不同：

```text
泄漏：修代码。
容量不足：扩容、限流、连接池、缓存、优化慢依赖。
```

---

## 六、常见问题

### 1. TIME_WAIT 要不要直接清理？

不要。先分析短连接来源和连接复用。

### 2. CLOSE_WAIT 多说明什么？

通常说明本地程序没有关闭连接。

### 3. fd 上限调大就解决了吗？

不一定。调大只能缓解，连接泄漏必须修代码。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 查看 fd 限制和进程 fd 数。
- 查看临时端口范围。
- 分析 TIME_WAIT 和 CLOSE_WAIT。
- 从 Go 代码中定位连接泄漏风险。
