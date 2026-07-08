# 2. HTTP 跳转 HTTPS 与 X-Forwarded-Proto

本节目标：配置 HTTP 自动跳转 HTTPS，并理解为什么 Nginx 终止 TLS 后还要把原始协议传给 Go。

---

## 一、为什么要 HTTP 跳 HTTPS

用户可能输入：

```text
http://api.example.com/api/ping
```

生产环境通常希望它自动跳到：

```text
https://api.example.com/api/ping
```

这样可以保证用户最终使用加密连接。

---

## 二、HTTP server 只负责跳转

```nginx
server {
    listen 80;
    server_name api.local;

    return 301 https://$host$request_uri;
}
```

解释：

- `301`：永久重定向。
- `$host`：保留原始域名。
- `$request_uri`：保留路径和查询参数。

请求：

```text
http://api.local/api/users?id=1
```

跳转：

```text
https://api.local/api/users?id=1
```

---

## 三、HTTPS server 负责代理

```nginx
server {
    listen 443 ssl;
    server_name api.local;

    ssl_certificate /home/your-user/nginx-lab/certs/api.local.crt;
    ssl_certificate_key /home/your-user/nginx-lab/certs/api.local.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## 四、为什么要 X-Forwarded-Proto

链路是：

```text
Client --HTTPS--> Nginx --HTTP--> Go
```

Go 服务收到的是 Nginx 发来的 HTTP 请求。

如果 Go 需要生成外部链接，例如：

```text
https://api.local/callback
```

它不能只看自己收到的协议，因为它看到的是 HTTP。

所以 Nginx 要告诉 Go：

```text
X-Forwarded-Proto: https
```

---

## 五、Go 中读取协议

```go
proto := r.Header.Get("X-Forwarded-Proto")
if proto == "" {
    proto = "http"
}
```

常见用途：

- 生成回调 URL。
- OAuth 跳转。
- 判断是否安全请求。
- 写访问日志。

---

## 六、验证跳转

```bash
curl -I http://api.local/api/ping
```

期望看到：

```text
HTTP/1.1 301 Moved Permanently
Location: https://api.local/api/ping
```

验证 HTTPS：

```bash
curl -k https://api.local/api/headers
```

期望看到：

```json
"x_forwarded_proto": "https"
```

---

## 七、常见问题

### 1. 跳转后路径丢了

错误写法：

```nginx
return 301 https://$host;
```

这样不会保留 `/api/ping`。

正确写法：

```nginx
return 301 https://$host$request_uri;
```

### 2. Go 生成 http 链接

检查是否设置：

```nginx
proxy_set_header X-Forwarded-Proto https;
```

### 3. 出现重复跳转

常见原因：

- Nginx 前面还有一层负载均衡。
- 上一层已经做 HTTPS，但传给 Nginx 是 HTTP。
- 应用层也做了跳转，判断协议不准。

这种多层代理场景要统一 `X-Forwarded-Proto` 的来源。

---

## 八、本节练习

1. 配置 HTTP server 只做 301 跳转。
2. 配置 HTTPS server 代理 Go。
3. 用 `curl -I` 验证 Location。
4. 访问 `/api/headers`，确认 Go 收到 `https`。
5. 故意去掉 `$request_uri`，观察路径丢失。

---

## 九、本节复盘

请确认你能回答：

1. 为什么 HTTP server 可以只写一行 return？
2. `$request_uri` 为什么重要？
3. TLS 终止是什么意思？
4. Go 为什么需要 `X-Forwarded-Proto`？
5. 多层代理时为什么容易出现重复跳转？

