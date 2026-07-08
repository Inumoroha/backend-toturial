# 03 性能调优：系统瓶颈、Go pprof 与定位方法

## 本节目标

性能问题不能只调 Nginx。你要能判断瓶颈在：

- Nginx。
- Go 服务。
- 数据库。
- 系统资源。
- 网络连接。

## 一、先建立基线

压测前记录：

```bash
nginx -v
go version
ulimit -n
cat /proc/cpuinfo | grep processor | wc -l
free -h
df -h
```

压测时记录：

- QPS。
- 平均延迟。
- P95、P99 延迟。
- 错误率。
- CPU。
- 内存。
- 连接数。

没有基线，就无法判断调优是否有效。

## 二、判断瓶颈位置

### 直连 Go 慢，Nginx 代理也慢

大概率瓶颈在 Go 或 Go 的依赖。

检查：

- Go handler 是否慢。
- 数据库是否慢。
- Redis 是否慢。
- 外部 HTTP 调用是否慢。

### 直连 Go 快，经过 Nginx 慢

可能是：

- Nginx worker 或连接限制。
- TLS 开销。
- gzip 开销。
- 日志磁盘 I/O。
- Nginx 到 upstream 连接没有复用。

### 只有高并发时慢

可能是：

- 文件描述符不足。
- Go 数据库连接池太小。
- Go goroutine 堆积。
- 上游服务连接池不足。
- 机器 CPU 或内存不足。

## 三、Go pprof 基础

在 Go 服务中加入：

```go
import _ "net/http/pprof"
```

如果你使用默认 mux，可以启动调试端口：

```go
go func() {
	log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
}()
```

只监听 `127.0.0.1`，不要暴露公网。

查看：

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/profile?seconds=30
```

常用：

```text
top
list 函数名
web
```

## 四、Nginx 日志辅助定位

如果日志显示：

```text
rt=1.205 urt=1.200
```

说明耗时基本在 upstream，也就是 Go 或 Go 的依赖。

如果：

```text
uct=0.800
```

说明连接 upstream 就慢，可能是连接压力或后端 accept 不及时。

如果：

```text
urt=-
```

说明请求可能没有成功到达 upstream，比如静态资源、限流、Nginx 直接返回或连接失败。

## 五、常见系统参数观察

连接统计：

```bash
ss -s
```

大量 TIME_WAIT 不一定是问题，但要结合端口耗尽、短连接压力一起看。

文件描述符：

```bash
ulimit -n
cat /proc/$(pgrep -o nginx)/limits | grep "open files"
```

磁盘 I/O：

```bash
iostat -x 1
```

如果没有 `iostat`，需要安装 `sysstat`。

## 六、调优原则

1. 先复现问题。
2. 再定位瓶颈。
3. 每次只改一个变量。
4. 修改后重新压测。
5. 记录结果。
6. 能通过架构解决的，不要只靠调参数。

例如数据库慢查询导致 504，调大 Nginx 超时只是让用户等更久，不是根治。

## 七、本节练习

1. 对 Go 直连和 Nginx 代理分别压测。
2. 在 access log 中加入 upstream 耗时。
3. 写一个 CPU 密集接口，用 pprof 找热点。
4. 写一个 sleep 慢接口，观察 Nginx 日志。
5. 修改 upstream keepalive，再压测比较。

## 八、你应该掌握

学完本节，你应该能用数据和日志判断性能瓶颈，而不是凭感觉修改 Nginx 配置。

