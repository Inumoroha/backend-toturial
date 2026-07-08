# 01 性能调优：Nginx 性能模型与系统限制

## 本节目标

理解 Nginx 为什么性能高，以及常见性能参数到底影响什么。调优不是照抄配置，而是先知道瓶颈在哪里。

## 一、Nginx 的基本模型

Nginx 通常由：

- 一个 master 进程。
- 多个 worker 进程。

master 负责管理 worker，worker 负责处理连接和请求。

查看进程：

```bash
ps aux | grep nginx
```

你会看到类似：

```text
nginx: master process
nginx: worker process
```

## 二、worker_processes

```nginx
worker_processes auto;
```

`auto` 通常会根据 CPU 核心数设置 worker 数量。大多数场景使用 `auto` 即可。

## 三、worker_connections

```nginx
events {
    worker_connections 10240;
}
```

理论最大连接数大致是：

```text
worker_processes * worker_connections
```

但实际还会受系统文件描述符限制影响。

## 四、文件描述符限制

查看：

```bash
ulimit -n
```

systemd 服务还可能有自己的限制：

```bash
systemctl show nginx | grep LimitNOFILE
```

如果连接量很高，但文件描述符限制太低，可能出现连接失败。

## 五、keepalive_timeout

```nginx
keepalive_timeout 65;
```

HTTP keep-alive 可以复用连接，减少 TCP 握手成本。

权衡：

- 太短：连接复用收益低。
- 太长：空闲连接占用资源。

学习阶段保持默认或 65 秒即可。

## 六、sendfile 与 tcp_nopush

静态文件服务常用：

```nginx
sendfile on;
tcp_nopush on;
```

`sendfile` 可以减少文件传输时的数据拷贝，提高静态文件性能。

## 七、上游连接复用

代理 Go 服务时，可以在 upstream 中配置 keepalive：

```nginx
upstream go_api {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    listen 80;
    server_name api.local;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_pass http://go_api;
    }
}
```

作用：Nginx 与 Go upstream 之间复用连接，减少频繁建连。

## 八、基础性能模板

```nginx
worker_processes auto;

events {
    worker_connections 10240;
}

http {
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    upstream go_api {
        server 127.0.0.1:8080;
        keepalive 32;
    }

    server {
        listen 80;
        server_name api.local;

        location / {
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_pass http://go_api;
        }
    }
}
```

## 九、本节练习

1. 查看 Nginx master 和 worker 进程。
2. 修改 `worker_processes auto`。
3. 查看系统文件描述符限制。
4. 配置 upstream keepalive。
5. 对比配置前后连接表现。

## 十、你应该掌握

学完本节，你应该能解释：

- worker 进程和连接数的关系。
- 系统限制为什么会影响 Nginx。
- keep-alive 如何减少建连成本。
- 为什么性能调优必须结合压测和监控。

