# 02 性能调优：压测、观察与调参

## 本节目标

学习如何用压测验证 Nginx 与 Go 服务的性能，而不是凭感觉调配置。

## 一、压测前的原则

压测要回答具体问题：

- Go 服务直连能扛多少 QPS？
- 经过 Nginx 后性能变化多少？
- 瓶颈在 CPU、内存、连接数、磁盘还是后端依赖？
- 慢请求是 Nginx 造成的，还是 Go 服务造成的？

不要只看 QPS，还要看错误率和延迟。

## 二、可选压测工具

常见工具：

- `wrk`
- `hey`
- `ab`

示例使用 `hey`：

```bash
hey -n 10000 -c 100 http://127.0.0.1:8080/api/ping
hey -n 10000 -c 100 http://api.local/api/ping
```

参数：

- `-n 10000`：总请求数。
- `-c 100`：并发数。

## 三、直连 Go 与经过 Nginx 对比

先压 Go：

```bash
hey -n 10000 -c 100 http://127.0.0.1:8080/api/ping
```

再压 Nginx：

```bash
hey -n 10000 -c 100 http://api.local/api/ping
```

观察：

- QPS 是否明显下降。
- 平均延迟是否明显增加。
- 95%、99% 延迟是否异常。
- 是否出现非 2xx。

如果 Nginx 后延迟只略微增加，通常是正常的。

## 四、压测时观察系统

CPU：

```bash
top
htop
```

连接：

```bash
ss -s
ss -ant | wc -l
```

端口：

```bash
ss -lntp
```

日志：

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

Go 服务：

- 看应用日志。
- 看数据库连接池。
- 看外部接口耗时。
- 必要时使用 pprof。

## 五、常见瓶颈判断

### CPU 打满

可能原因：

- Go 业务逻辑重。
- JSON 序列化压力大。
- gzip 压缩等级过高。
- TLS 握手压力大。

### 连接数异常

可能原因：

- keep-alive 设置不合理。
- 客户端大量短连接。
- 文件描述符限制太低。
- 后端连接池不足。

### 5xx 增加

可能原因：

- Go 服务处理不过来。
- upstream 连接失败。
- 超时配置触发。
- 限流或连接限制触发。

## 六、调优顺序建议

1. 先保证 Go 服务本身性能健康。
2. 再确认 Nginx 配置正确。
3. 再观察系统限制。
4. 最后才调 worker、连接、keepalive、gzip 等参数。

不要在没有压测和日志的情况下盲目调大所有参数。

## 七、本节练习

1. 用 `hey` 压测 Go 直连。
2. 用 `hey` 压测 Nginx 代理。
3. 记录 QPS、平均延迟、P95、P99。
4. 打开 gzip，再压一次。
5. 配置 upstream keepalive，再压一次。

## 八、你应该掌握

学完本节，你应该会用数据回答：

- Nginx 是否是瓶颈。
- 慢请求主要来自哪一层。
- 某个配置调整是否真的带来收益。

