# 01 HTTPS 与安全：证书、TLS 终止与跳转

## 本节目标

掌握用 Nginx 给 Go 服务提供 HTTPS 访问。生产环境常见模式是：

```text
Client --HTTPS--> Nginx --HTTP--> Go API
```

Nginx 负责 TLS 终止，Go 服务只监听本机或内网端口。

## 一、HTTPS 解决什么问题

HTTPS 主要提供：

- 加密：请求内容不容易被中间人读取。
- 身份验证：浏览器能确认服务端证书属于目标域名。
- 完整性：传输内容不容易被篡改。

对后端工程师来说，你至少要知道证书和私钥是不同文件，私钥不能泄露。

## 二、使用自签名证书练习

学习阶段可以先用自签名证书。

```bash
mkdir -p ~/nginx-lab/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ~/nginx-lab/certs/api.local.key \
  -out ~/nginx-lab/certs/api.local.crt \
  -subj "/CN=api.local"
```

这会生成：

```text
api.local.crt
api.local.key
```

浏览器会提示不受信任，这是正常的，因为它不是受信任 CA 签发的。

## 三、配置 HTTPS

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

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

测试：

```bash
curl -k https://api.local/api/ping
```

`-k` 表示忽略自签名证书的不受信任提示。

## 四、HTTP 跳转 HTTPS

生产环境通常把 HTTP 自动跳转到 HTTPS。

```nginx
server {
    listen 80;
    server_name api.local;

    return 301 https://$host$request_uri;
}
```

完整结构：

```nginx
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

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

测试跳转：

```bash
curl -I http://api.local/api/ping
```

你应该能看到 `301` 和 `Location: https://...`。

## 五、真实证书

真实公网域名推荐使用 Let's Encrypt 和 Certbot。

常见流程：

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
```

证书通常位于：

```text
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
```

实际路径以 Certbot 输出为准。

## 六、Go 服务如何感知 HTTPS

Nginx 到 Go 之间可能是 HTTP，但用户访问的是 HTTPS，所以要传：

```nginx
proxy_set_header X-Forwarded-Proto https;
```

Go 服务里可以读取：

```go
proto := r.Header.Get("X-Forwarded-Proto")
```

这对生成回调 URL、OAuth 跳转地址、外部链接很重要。

## 七、本节练习

1. 生成自签名证书。
2. 配置 `443 ssl`。
3. 使用 `curl -k` 访问 HTTPS。
4. 配置 HTTP 跳转 HTTPS。
5. 在 Go 服务中读取 `X-Forwarded-Proto`。

## 八、你应该掌握

学完本节，你应该能：

- 给 Go 服务加上 HTTPS 访问入口。
- 解释 TLS 终止是什么意思。
- 配置 HTTP 到 HTTPS 的跳转。
- 知道证书和私钥的基本作用。

