# 03 项目实战：HTTPS、限流与安全网关

## 项目目标

在前面 Go API 代理的基础上，加入 HTTPS、HTTP 跳转 HTTPS、基础安全头、限流和上传限制。

最终效果：

```text
Client --HTTPS--> Nginx Gateway -> Go API
```

## 一、功能要求

Nginx 需要提供：

- `80` 自动跳转 `443`。
- `443 ssl` HTTPS 访问。
- 基础安全响应头。
- `/api/` 限流。
- `/api/upload` 单独设置上传大小。
- 代理到 Go 服务。

## 二、生成自签名证书

```bash
mkdir -p ~/nginx-lab/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ~/nginx-lab/certs/api.local.key \
  -out ~/nginx-lab/certs/api.local.crt \
  -subj "/CN=api.local"
```

## 三、Nginx 配置

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    server_name api.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.local;

    ssl_certificate /home/your-user/nginx-lab/certs/api.local.crt;
    ssl_certificate_key /home/your-user/nginx-lab/certs/api.local.key;

    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header Referrer-Policy no-referrer-when-downgrade always;

    client_max_body_size 10m;
    limit_req_status 429;

    location /api/upload {
        client_max_body_size 100m;
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;

        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

## 四、验证命令

测试跳转：

```bash
curl -I http://api.local/api/ping
```

测试 HTTPS：

```bash
curl -k https://api.local/api/ping
```

测试安全头：

```bash
curl -k -I https://api.local/api/ping
```

测试限流：

```bash
for i in {1..100}; do curl -k -s -o /dev/null -w "%{http_code}\n" https://api.local/api/ping; done
```

## 五、验收标准

- HTTP 请求会 301 跳转到 HTTPS。
- HTTPS 能成功访问 Go API。
- 响应头包含安全头。
- 高频请求会被限制，返回 429。
- 上传接口有单独大小限制。

## 六、复盘问题

1. 为什么 Go 服务仍然可以只监听 HTTP？
2. `X-Forwarded-Proto` 为什么要设置为 `https`？
3. 为什么 HSTS 不建议一开始就随便开长时间？

