# 02 HTTPS 与安全：基础加固配置

## 本节目标

学习 Nginx 常见安全加固项。安全配置不应该盲目复制，应该理解每个配置解决什么问题。

## 一、隐藏 Nginx 版本号

在 `http` 块中配置：

```nginx
server_tokens off;
```

作用：减少响应头和错误页中暴露的 Nginx 版本信息。

## 二、常用安全响应头

```nginx
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy no-referrer-when-downgrade always;
```

含义：

- `X-Content-Type-Options`：减少 MIME 嗅探风险。
- `X-Frame-Options`：限制页面被 iframe 嵌入，降低点击劫持风险。
- `Referrer-Policy`：控制 Referer 信息泄露。

如果是纯 API 服务，安全头的收益有限但仍可配置。

## 三、HSTS

HSTS 告诉浏览器以后强制使用 HTTPS。

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

注意：确认 HTTPS 永久可用后再给正式域名开启长时间 HSTS。否则配置错误时，浏览器会继续强制访问 HTTPS。

## 四、限制请求方法

如果某些路径只允许 GET：

```nginx
location /public/ {
    limit_except GET {
        deny all;
    }
}
```

API 服务通常由 Go 处理方法限制，Nginx 只做粗粒度保护。

## 五、IP 访问控制

管理后台只允许内网访问：

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    proxy_pass http://127.0.0.1:8080;
}
```

注意：如果 Nginx 前面还有一层负载均衡，`allow` 判断的可能不是用户真实 IP，而是上一层代理 IP。

## 六、Basic Auth

适合给临时后台、测试环境、文档站加一层简单认证。

安装工具：

```bash
sudo apt install -y apache2-utils
```

创建账号：

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

配置：

```nginx
location /admin/ {
    auth_basic "Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://127.0.0.1:8080;
}
```

## 七、完整安全模板

```nginx
server_tokens off;

server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header Referrer-Policy no-referrer-when-downgrade always;
    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

## 八、本节练习

1. 开启 `server_tokens off`。
2. 添加基础安全响应头。
3. 给 `/admin/` 配置 Basic Auth。
4. 给 `/admin/` 加 IP 访问限制。
5. 思考哪些安全逻辑应该放在 Go 服务中。

## 九、你应该掌握

学完本节，你应该知道：

- Nginx 能做哪些基础安全加固。
- 哪些安全能力适合 Nginx，哪些应该放在业务服务。
- HSTS 为什么要谨慎开启。

