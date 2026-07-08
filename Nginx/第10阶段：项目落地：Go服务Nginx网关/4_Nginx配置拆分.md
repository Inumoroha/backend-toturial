# 4. Nginx 配置拆分

本节目标：把最终项目的 Nginx 配置拆成多个文件，形成可维护的工程结构。

---

## 一、为什么要拆分配置

不要把所有内容都写进一个巨大 `nginx.conf`。拆分后更清楚：

```text
00-log-format.conf   日志格式
10-upstream.conf     Go upstream
20-app.conf          server 和 location
```

这和 PostgreSQL 项目中拆迁移文件、Go 项目中拆 handler/service/repository 是同一个思路：职责清楚。

---

## 二、主配置 nginx.conf

`nginx/nginx.conf`：

```nginx
worker_processes auto;

events {
    worker_connections 10240;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    include /etc/nginx/conf.d/*.conf;
}
```

---

## 三、日志格式

`nginx/conf.d/00-log-format.conf`：

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

---

## 四、upstream 配置

`nginx/conf.d/10-upstream.conf`：

```nginx
upstream go_api {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8081 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=10s;
    keepalive 32;
}
```

---

## 五、应用 server 配置

`nginx/conf.d/20-app.conf`：

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    server_name app.local;

    access_log /var/log/nginx/app_access.log api_main;
    error_log /var/log/nginx/app_error.log warn;

    root /var/www/app/static;
    index index.html;

    limit_req_status 429;

    location = /health {
        return 200 "ok\n";
    }

    location ^~ /assets/ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    location /api/upload {
        client_max_body_size 100m;
        proxy_pass http://go_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header X-Request-ID $request_id;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;

        proxy_pass http://go_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header X-Request-ID $request_id;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 六、部署到系统 Nginx

如果不用 Docker，可以把文件复制到：

```text
/etc/nginx/nginx.conf
/etc/nginx/conf.d/00-log-format.conf
/etc/nginx/conf.d/10-upstream.conf
/etc/nginx/conf.d/20-app.conf
```

静态文件放到：

```text
/var/www/app/static
```

检查：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 七、本节练习

1. 创建 `nginx/nginx.conf`。
2. 创建 `00-log-format.conf`。
3. 创建 `10-upstream.conf`。
4. 创建 `20-app.conf`。
5. 使用 `nginx -T` 确认所有配置被加载。
6. 请求 `/`、`/assets/style.css`、`/api/ping`。

---

## 八、本节复盘

请确认你能回答：

1. 为什么日志格式要单独拆出来？
2. upstream 为什么适合单独一个文件？
3. `/api/upload` 为什么要放在 `/api/` 前面单独写？
4. `try_files $uri $uri/ /index.html` 在项目中解决什么问题？
5. `nginx -T` 在配置拆分后为什么更重要？

