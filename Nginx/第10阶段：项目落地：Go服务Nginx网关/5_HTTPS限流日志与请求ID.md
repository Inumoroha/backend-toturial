# 5. HTTPS、限流、日志与请求 ID

本节目标：在项目配置中补齐 HTTPS、HTTP 跳转 HTTPS、限流、请求 ID 和 upstream 耗时日志。

---

## 一、生成自签名证书

```bash
mkdir -p nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/app.local.key \
  -out nginx/certs/app.local.crt \
  -subj "/CN=app.local"
```

学习环境用自签名证书即可。生产环境使用 Let's Encrypt 或公司证书。

---

## 二、HTTP 跳 HTTPS

```nginx
server {
    listen 80;
    server_name app.local;

    return 301 https://$host$request_uri;
}
```

验证：

```bash
curl -I http://app.local/api/ping
```

期望：

```text
301
Location: https://app.local/api/ping
```

---

## 三、HTTPS server

把原来的 `listen 80` server 改成：

```nginx
server {
    listen 443 ssl;
    server_name app.local;

    ssl_certificate /etc/nginx/certs/app.local.crt;
    ssl_certificate_key /etc/nginx/certs/app.local.key;

    access_log /var/log/nginx/app_access.log api_main;
    error_log /var/log/nginx/app_error.log warn;

    root /var/www/app/static;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

后续把 API location 放回这个 HTTPS server 中。

---

## 四、请求 ID

Nginx 日志中：

```nginx
log_format api_main 'rid=$request_id ...';
```

传给 Go：

```nginx
proxy_set_header X-Request-ID $request_id;
```

Go 日志中打印：

```text
request_id=xxx
```

这样就能关联 Nginx 与 Go。

---

## 五、限流

`http` 上下文：

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
```

server 中：

```nginx
limit_req_status 429;
```

API location 中：

```nginx
limit_req zone=api_limit burst=20 nodelay;
```

测试：

```bash
for i in {1..100}; do curl -k -s -o /dev/null -w "%{http_code}\n" https://app.local/api/ping; done
```

---

## 六、完整 API location

```nginx
location /api/ {
    limit_req zone=api_limit burst=20 nodelay;

    proxy_pass http://go_api;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header X-Request-ID $request_id;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    proxy_connect_timeout 5s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

注意 HTTPS server 中建议直接设置：

```nginx
proxy_set_header X-Forwarded-Proto https;
```

而不是让 Go 猜。

---

## 七、验证

HTTPS：

```bash
curl -k https://app.local/api/ping
```

请求头：

```bash
curl -k https://app.local/api/headers
```

日志：

```bash
sudo tail -n 20 /var/log/nginx/app_access.log
```

Go 控制台日志中查同一个 request_id。

---

## 八、本节练习

1. 生成自签名证书。
2. 配置 HTTP 跳 HTTPS。
3. 配置 HTTPS server。
4. 在日志中记录 request_id。
5. 把 request_id 传给 Go。
6. 触发限流并观察 429。

---

## 九、本节复盘

请确认你能回答：

1. 为什么 HTTP server 只做跳转？
2. 为什么 HTTPS 场景要传 `X-Forwarded-Proto: https`？
3. 请求 ID 如何贯穿 Nginx 和 Go？
4. 限流为什么建议返回 429？
5. 如何验证限流真的生效？

