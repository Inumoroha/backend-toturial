# 02 反向代理 Go 服务：请求头、超时与 WebSocket

## 本节目标

在基础代理上补齐生产中最常见的代理配置：

- 透传原始 Host。
- 传递真实客户端 IP。
- 传递原始协议。
- 配置连接和读取超时。
- 支持 WebSocket。

## 一、为什么要设置请求头

如果只写：

```nginx
proxy_pass http://127.0.0.1:8080;
```

Go 服务能收到请求，但很多原始信息可能不完整。真实项目里，Go 服务经常需要：

- 记录真实客户端 IP。
- 判断请求原本是 HTTP 还是 HTTPS。
- 生成回调 URL。
- 做审计和安全日志。

## 二、推荐基础代理头

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

含义：

- `Host`：保留客户端访问的域名。
- `X-Real-IP`：记录直接连接 Nginx 的客户端 IP。
- `X-Forwarded-For`：记录代理链路上的客户端 IP 列表。
- `X-Forwarded-Proto`：记录原始协议，常见值是 `http` 或 `https`。

## 三、Go 中查看代理头

示例：

```go
http.HandleFunc("/api/headers", func(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/plain")
	w.Write([]byte("RemoteAddr: " + r.RemoteAddr + "\n"))
	w.Write([]byte("Host: " + r.Host + "\n"))
	w.Write([]byte("X-Real-IP: " + r.Header.Get("X-Real-IP") + "\n"))
	w.Write([]byte("X-Forwarded-For: " + r.Header.Get("X-Forwarded-For") + "\n"))
	w.Write([]byte("X-Forwarded-Proto: " + r.Header.Get("X-Forwarded-Proto") + "\n"))
})
```

通过 Nginx 访问：

```bash
curl http://api.local/api/headers
```

你会发现 `RemoteAddr` 往往是 Nginx 地址，而真实客户端信息在代理头中。

## 四、超时配置

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;

    proxy_connect_timeout 5s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

含义：

- `proxy_connect_timeout`：Nginx 连接后端的超时时间。
- `proxy_send_timeout`：Nginx 向后端发送请求的超时时间。
- `proxy_read_timeout`：Nginx 等待后端响应的超时时间。

Go 后端慢查询、外部接口卡住、锁竞争等问题，常会导致 `proxy_read_timeout` 触发，最终返回 504。

## 五、请求体大小限制

```nginx
server {
    client_max_body_size 10m;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

如果上传文件超过限制，Nginx 会返回 `413 Request Entity Too Large`。

建议：

- 普通 API：可以设小一点，例如 `1m` 或 `2m`。
- 文件上传接口：单独 location 设置，例如 `50m`。

## 六、WebSocket 代理

WebSocket 需要升级协议。

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;
    server_name ws.local;

    location /ws/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
    }
}
```

如果你的 Go 项目使用 WebSocket，缺少这些配置时，经常会握手失败或连接很快断开。

## 七、完整 API 代理模板

```nginx
server {
    listen 80;
    server_name api.local;

    client_max_body_size 10m;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

## 八、本节练习

1. 在 Go 服务中打印代理头。
2. 配置 `X-Real-IP` 和 `X-Forwarded-For`。
3. 写一个慢接口，睡眠 35 秒，观察 504。
4. 调整 `proxy_read_timeout` 后再次测试。
5. 配置一个上传大小限制，并用大文件测试 413。

## 九、你应该掌握

学完本节，你应该知道：

- Go 服务为什么不能直接信任 `RemoteAddr`。
- 代理头分别解决什么问题。
- 504 和超时配置有什么关系。
- WebSocket 为什么需要特殊请求头。

