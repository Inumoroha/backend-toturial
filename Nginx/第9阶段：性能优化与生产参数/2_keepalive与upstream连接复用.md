# 2. keepalive 与 upstream 连接复用

本节目标：理解客户端到 Nginx、Nginx 到 Go 之间的连接复用，并掌握 upstream keepalive 的基础配置。

---

## 一、为什么连接复用重要

如果每个请求都重新建立 TCP 连接，成本会更高：

```text
建立连接
发送请求
返回响应
关闭连接
```

keep-alive 可以复用连接：

```text
建立连接
请求 1
请求 2
请求 3
关闭连接
```

这能减少 TCP 握手和连接管理开销。

---

## 二、客户端到 Nginx 的 keepalive

配置：

```nginx
http {
    keepalive_timeout 65;
}
```

含义：客户端连接空闲多久后关闭。

权衡：

- 太短：连接复用效果差。
- 太长：空闲连接占用资源。

大多数场景保持默认或 65 秒即可。

---

## 三、Nginx 到 Go 的 upstream keepalive

配置：

```nginx
upstream go_api {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
    keepalive 32;
}

server {
    listen 80;
    server_name api.local;

    location /api/ {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_pass http://go_api;
    }
}
```

关键点：

- `keepalive 32` 写在 upstream 中。
- `proxy_http_version 1.1` 使用 HTTP/1.1。
- `proxy_set_header Connection ""` 避免关闭 upstream 连接。

---

## 四、Go 服务也要支持合理超时

Go 中建议配置：

```go
server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  15 * time.Second,
    WriteTimeout: 60 * time.Second,
    IdleTimeout:  60 * time.Second,
}
```

Nginx 和 Go 的连接行为要一起看。

---

## 五、如何观察连接

查看连接统计：

```bash
ss -s
```

查看到 Go 端口的连接：

```bash
ss -ant | grep ':8080'
```

压测前后对比连接数量：

```bash
hey -n 10000 -c 100 http://api.local/api/ping
```

---

## 六、什么时候不应盲目加大 keepalive

不要把 keepalive 数值无限调大。它会占用连接资源。

需要结合：

- worker 数量。
- upstream 实例数。
- Go 服务最大连接承受能力。
- 文件描述符限制。
- 实际并发。

---

## 七、本节练习

1. 配置 upstream keepalive。
2. 压测前查看连接数。
3. 压测后查看连接数。
4. 去掉 upstream keepalive，再压一次对比。
5. 观察 Go 服务日志和 Nginx error log 是否异常。

---

## 八、本节复盘

请确认你能回答：

1. keep-alive 解决什么问题？
2. `keepalive_timeout` 控制哪一段连接？
3. upstream `keepalive` 控制哪一段连接？
4. 为什么要设置 `proxy_http_version 1.1`？
5. 为什么 keepalive 不是越大越好？

