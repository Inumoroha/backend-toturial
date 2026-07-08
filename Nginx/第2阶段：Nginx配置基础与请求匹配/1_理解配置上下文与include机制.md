# 1. 理解配置上下文与 include 机制

本节目标：理解 Nginx 配置为什么有层级，哪些指令应该写在哪里，以及如何确认某个配置文件是否真的被加载。

Nginx 配置文件不是简单的键值对，而是一种有上下文的结构。很多新手报错：

```text
"server" directive is not allowed here
```

本质原因就是指令写错了位置。

---

## 一、Nginx 配置的基本结构

典型结构：

```nginx
user www-data;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;

    server {
        listen 80;
        server_name example.local;

        location / {
            return 200 "hello\n";
        }
    }
}
```

可以理解为：

```text
main
  events
  http
    server
      location
```

---

## 二、常见上下文说明

### main

最外层，不写在任何 `{}` 里面。

常见指令：

- `user`
- `worker_processes`
- `error_log`
- `pid`

### events

连接事件处理相关。

常见指令：

- `worker_connections`

### http

HTTP 服务相关配置。

常见指令：

- `include`
- `log_format`
- `access_log`
- `upstream`
- `server`
- `gzip`
- `limit_req_zone`

### server

一个虚拟主机。

常见指令：

- `listen`
- `server_name`
- `location`
- `root`
- `access_log`
- `client_max_body_size`

### location

一个 URI 处理规则。

常见指令：

- `return`
- `root`
- `alias`
- `try_files`
- `proxy_pass`
- `proxy_set_header`
- `limit_req`

---

## 三、常见指令位置表

| 指令 | 常见上下文 |
| --- | --- |
| `worker_processes` | main |
| `events` | main |
| `http` | main |
| `server` | http |
| `location` | server |
| `upstream` | http |
| `log_format` | http |
| `limit_req_zone` | http |
| `limit_req` | http、server、location |
| `proxy_pass` | location |
| `root` | http、server、location |
| `alias` | location |

如果不确定某个指令写在哪里，去官方文档看 `Context`。

---

## 四、错误示例：server 写错位置

错误配置：

```nginx
events {
    server {
        listen 80;
        server_name wrong.local;
    }
}
```

检查：

```bash
sudo nginx -t
```

会报错：

```text
"server" directive is not allowed here
```

正确写法：

```nginx
http {
    server {
        listen 80;
        server_name right.local;
    }
}
```

---

## 五、include 机制

主配置里通常有：

```nginx
include /etc/nginx/conf.d/*.conf;
```

意思是：加载这个目录下所有 `.conf` 文件。

Ubuntu 还可能有：

```nginx
include /etc/nginx/sites-enabled/*;
```

这就是为什么你在 `/etc/nginx/conf.d/api.conf` 写 server，也能被 Nginx 识别。

---

## 六、如何确认配置被加载

使用：

```bash
sudo nginx -T
```

查找你的配置：

```bash
sudo nginx -T | grep -A 20 api.local
```

如果看不到，检查：

- 文件路径是否正确。
- 文件后缀是否是 `.conf`。
- 主配置是否 include 了这个目录。
- 文件权限是否允许 Nginx 读取。

---

## 七、推荐配置拆分方式

学习阶段可以这样组织：

```text
/etc/nginx/conf.d/
  00-log-format.conf
  10-upstream-go-api.conf
  20-api.local.conf
  30-static.local.conf
```

命名建议：

- `00` 放日志格式、map 等全局 HTTP 配置。
- `10` 放 upstream。
- `20`、`30` 放具体 server。

这样看起来就像 PostgreSQL 迁移文件一样，有顺序、有职责。

---

## 八、本节练习

1. 画出 main、events、http、server、location 层级。
2. 故意把 server 写进 events，观察报错。
3. 创建 `/etc/nginx/conf.d/20-test.local.conf`。
4. 使用 `nginx -T` 确认配置被加载。
5. 把文件改成 `.txt` 后缀，再观察是否加载。

---

## 九、本节复盘

请确认你能回答：

1. `server` 应该写在哪个上下文？
2. `upstream` 为什么不能写在 `server` 里面？
3. `include /etc/nginx/conf.d/*.conf` 做了什么？
4. 配置没生效时为什么要先看 `nginx -T`？
5. 数字前缀拆分配置有什么好处？

