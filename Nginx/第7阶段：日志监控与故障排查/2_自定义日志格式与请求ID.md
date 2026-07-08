# 2. 自定义日志格式与请求 ID

本节目标：给 Nginx access log 增加 upstream 耗时和请求 ID，并把请求 ID 传给 Go 服务，实现 Nginx 日志和 Go 日志关联。

---

## 一、为什么需要请求 ID

一次请求可能经过：

```text
Nginx -> Go API -> Redis -> PostgreSQL -> 第三方服务
```

如果没有请求 ID，排查问题只能靠时间和路径猜。

有请求 ID 后，你可以在 Nginx 日志中看到：

```text
rid=abc123
```

也可以在 Go 日志中看到：

```text
request_id=abc123
```

这样就能确认两条日志属于同一次请求。

---

## 二、自定义日志格式

在 `http` 上下文中配置：

```nginx
log_format api_main 'rid=$request_id '
                    'remote=$remote_addr '
                    'time=[$time_local] '
                    'request="$request" '
                    'status=$status '
                    'bytes=$body_bytes_sent '
                    'rt=$request_time '
                    'uct=$upstream_connect_time '
                    'uht=$upstream_header_time '
                    'urt=$upstream_response_time '
                    'upstream=$upstream_addr';
```

字段解释：

- `rid`：请求 ID。
- `rt`：Nginx 处理整个请求耗时。
- `uct`：连接 upstream 耗时。
- `uht`：收到 upstream 响应头耗时。
- `urt`：upstream 总响应耗时。
- `upstream`：实际 Go 实例地址。

---

## 三、使用日志格式

```nginx
server {
    listen 80;
    server_name api.local;

    access_log /var/log/nginx/api_access.log api_main;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header X-Request-ID $request_id;
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 四、Go 服务读取请求 ID

```go
requestID := r.Header.Get("X-Request-ID")
```

日志中打印：

```go
log.Printf("request_id=%s method=%s path=%s", requestID, r.Method, r.URL.Path)
```

如果你使用本教程项目中的 Go 服务，它已经会打印：

```text
request_id=xxx method=GET path=/api/ping
```

---

## 五、验证

请求：

```bash
curl -H "Host: api.local" http://127.0.0.1/api/ping
```

查看 Nginx 日志：

```bash
sudo tail -n 5 /var/log/nginx/api_access.log
```

查看 Go 日志。

你应该能在两边看到同一个请求 ID。

---

## 六、慢请求分析示例

Nginx 日志：

```text
rid=abc123 request="GET /api/slow?seconds=5 HTTP/1.1" status=200 rt=5.002 urt=5.001 upstream=127.0.0.1:8080
```

Go 日志：

```text
request_id=abc123 method=GET path=/api/slow?seconds=5 cost=5.001s
```

结论：

```text
耗时主要发生在 Go 服务内部。
```

---

## 七、本节练习

1. 配置 `api_main` 日志格式。
2. 在 access log 中加入 `$request_id`。
3. 通过 `X-Request-ID` 传给 Go。
4. 请求 `/api/ping`，关联 Nginx 和 Go 日志。
5. 请求 `/api/slow?seconds=5`，分析耗时字段。

---

## 八、本节复盘

请确认你能回答：

1. 请求 ID 解决什么问题？
2. `$request_time` 和 `$upstream_response_time` 有什么区别？
3. `$upstream_addr` 有什么用？
4. Nginx 如何把请求 ID 传给 Go？
5. 为什么慢请求排查要同时看 Nginx 和 Go 日志？

