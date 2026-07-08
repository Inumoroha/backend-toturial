# 1. worker、连接数与系统限制

本节目标：理解 Nginx worker 进程、连接数和系统文件描述符限制之间的关系。

---

## 一、查看 Nginx 进程

```bash
ps -ef | grep nginx
```

你会看到：

```text
nginx: master process
nginx: worker process
nginx: worker process
```

master 负责管理 worker，worker 负责处理请求。

---

## 二、worker_processes

```nginx
worker_processes auto;
```

`auto` 表示自动根据 CPU 核心数设置 worker 数。

大多数场景使用 `auto` 即可。

---

## 三、worker_connections

```nginx
events {
    worker_connections 10240;
}
```

它表示每个 worker 能处理的最大连接数。

理论最大连接数大致是：

```text
worker_processes * worker_connections
```

但这只是理论值，还受系统限制影响。

---

## 四、文件描述符限制

Linux 中连接、文件、socket 都会占用文件描述符。

查看当前 shell 限制：

```bash
ulimit -n
```

查看 Nginx 进程限制：

```bash
cat /proc/$(pgrep -o nginx)/limits | grep "open files"
```

systemd 中可能还要看：

```bash
systemctl show nginx | grep LimitNOFILE
```

---

## 五、连接数不够会怎样

可能现象：

- 请求失败。
- error log 出现资源不足。
- 高并发下连接建立很慢。
- Nginx 或 Go 服务出现大量错误。

但不要一看到性能问题就调大连接数。先压测和观察。

---

## 六、基础配置示例

```nginx
worker_processes auto;

events {
    worker_connections 10240;
}

http {
    keepalive_timeout 65;
}
```

如果确实需要更高并发，还要配合系统限制调整。

---

## 七、观察连接

连接统计：

```bash
ss -s
```

查看 80 端口连接：

```bash
ss -ant | grep ':80' | head
```

统计连接数量：

```bash
ss -ant | wc -l
```

---

## 八、本节练习

1. 查看 master 和 worker。
2. 设置 `worker_processes auto`。
3. 查看 `worker_connections`。
4. 查看 `ulimit -n`。
5. 查看 Nginx 进程 open files 限制。
6. 用压测工具提高并发，观察连接数变化。

---

## 九、本节复盘

请确认你能回答：

1. master 和 worker 分别负责什么？
2. `worker_processes auto` 的含义是什么？
3. `worker_connections` 表示总连接数吗？
4. 文件描述符为什么会限制并发？
5. 为什么不能盲目调大所有参数？

