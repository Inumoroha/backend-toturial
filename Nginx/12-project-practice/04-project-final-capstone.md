# 04 项目实战：Go + Nginx 综合部署

## 项目目标

完成一个接近真实生产环境的小型后端部署项目。这个项目用于检验你是否真正掌握 Nginx 在 Go 后端系统中的核心能力。

## 一、最终架构

```text
Client
  |
  | HTTPS
  v
Nginx
  |-- /              -> Static frontend
  |-- /assets/       -> Static assets with cache
  |-- /api/          -> Go API upstream
  |-- /api/upload    -> Larger body size
  |-- /health        -> Nginx health response
  |
  v
Go API instances
  |-- 127.0.0.1:8080
  |-- 127.0.0.1:8081
  |-- 127.0.0.1:8082
```

## 二、功能清单

你需要完成：

- 静态前端页面部署。
- Go API 三实例运行。
- Nginx upstream 负载均衡。
- HTTPS 访问。
- HTTP 自动跳转 HTTPS。
- 真实 IP 代理头。
- 自定义 access log。
- 502、504 可复现实验。
- `/api/` 基础限流。
- `/assets/` 静态缓存。
- `/api/upload` 上传大小限制。
- systemd 或 Docker Compose 部署方式二选一。

## 三、Go API 要求

接口：

- `GET /api/ping`
- `GET /api/headers`
- `GET /api/slow?seconds=5`
- `POST /api/upload`
- `GET /health`

`/api/slow` 用于测试超时，`/api/upload` 用于测试请求体大小。

## 四、Nginx 配置模板

```nginx
worker_processes auto;

events {
    worker_connections 10240;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format api_main '$remote_addr [$time_local] "$request" '
                        '$status $body_bytes_sent '
                        'rt=$request_time '
                        'uct=$upstream_connect_time '
                        'uht=$upstream_header_time '
                        'urt=$upstream_response_time '
                        'upstream=$upstream_addr';

    access_log /var/log/nginx/access.log api_main;
    error_log /var/log/nginx/error.log warn;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    upstream go_api {
        server 127.0.0.1:8080 max_fails=3 fail_timeout=10s;
        server 127.0.0.1:8081 max_fails=3 fail_timeout=10s;
        server 127.0.0.1:8082 max_fails=3 fail_timeout=10s;
        keepalive 32;
    }

    server {
        listen 80;
        server_name app.local;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name app.local;

        ssl_certificate /home/your-user/nginx-lab/certs/app.local.crt;
        ssl_certificate_key /home/your-user/nginx-lab/certs/app.local.key;

        add_header X-Content-Type-Options nosniff always;
        add_header X-Frame-Options SAMEORIGIN always;
        add_header Referrer-Policy no-referrer-when-downgrade always;

        root /home/your-user/nginx-lab/static;
        index index.html;

        location = /health {
            return 200 "ok\n";
        }

        location /assets/ {
            expires 30d;
            add_header Cache-Control "public, immutable";
            try_files $uri =404;
        }

        location /api/upload {
            client_max_body_size 100m;
            proxy_pass http://go_api;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
        }

        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;

            proxy_pass http://go_api;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;

            proxy_connect_timeout 5s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }

        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```

## 五、验收步骤

### 1. 静态页面

```bash
curl -k https://app.local/
curl -k -I https://app.local/assets/style.css
```

要求：

- 首页可访问。
- assets 有缓存响应头。

### 2. API 代理

```bash
curl -k https://app.local/api/ping
curl -k https://app.local/api/headers
```

要求：

- 能访问 Go API。
- 能看到代理头。

### 3. 负载均衡

```bash
for i in {1..12}; do curl -k -s https://app.local/api/ping; echo; done
```

要求：

- 请求能落到不同实例。

### 4. HTTPS 跳转

```bash
curl -I http://app.local/api/ping
```

要求：

- 返回 301。
- Location 指向 HTTPS。

### 5. 限流

```bash
for i in {1..100}; do curl -k -s -o /dev/null -w "%{http_code}\n" https://app.local/api/ping; done
```

要求：

- 高频请求出现 429。

### 6. 502 实验

停掉所有 Go 实例：

```bash
curl -k https://app.local/api/ping
sudo tail -n 20 /var/log/nginx/error.log
```

要求：

- 能复现 502。
- 能从 error log 解释原因。

### 7. 504 实验

请求慢接口：

```bash
curl -k "https://app.local/api/slow?seconds=40"
```

要求：

- 能复现 504。
- 能通过日志中的 upstream 响应时间解释原因。

## 六、最终复盘

完成后，请你写一份自己的复盘，至少回答：

1. 请求从浏览器到 Go 服务经历了哪些步骤？
2. Nginx 如何选择 `server` 和 `location`？
3. Nginx 如何把请求分发到多个 Go 实例？
4. 真实 IP 是如何传给 Go 服务的？
5. 502 和 504 的排查路径分别是什么？
6. 哪些逻辑适合放 Nginx，哪些必须放 Go？

## 七、完成标准

如果你能独立完成本项目，并能解释模板中每一段配置的作用，说明你已经具备 Go 后端工程师日常工作所需的 Nginx 基础能力。

