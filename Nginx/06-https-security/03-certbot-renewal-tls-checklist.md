# 03 HTTPS 与安全：Certbot 续期与 TLS 检查清单

## 本节目标

真实生产环境中，配置 HTTPS 只是第一步。证书会过期，你需要知道如何续期、验证和排查。

## 一、Let's Encrypt 证书有效期

Let's Encrypt 证书有效期通常较短，需要自动续期。Certbot 安装后一般会创建 systemd timer 或 cron 任务。

查看 timer：

```bash
systemctl list-timers | grep certbot
```

手动测试续期：

```bash
sudo certbot renew --dry-run
```

`--dry-run` 不会真正替换证书，适合验证续期流程是否能跑通。

## 二、续期后重载 Nginx

Certbot 通常会自动处理 Nginx reload。如果你自己管理证书，可以用 deploy hook：

```bash
sudo certbot renew --deploy-hook "systemctl reload nginx"
```

不要只续期证书文件却不 reload。Nginx 可能还在使用旧证书。

## 三、检查证书时间

```bash
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -dates
```

检查域名证书：

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
```

`-servername` 很重要，它用于 SNI。多个域名共用一个 IP 时，没有 SNI 可能拿到默认证书。

## 四、常见 HTTPS 问题

### 证书域名不匹配

现象：

```text
certificate is not valid for this name
```

原因：访问的是 `api.example.com`，证书却签给了 `example.com`。

### 证书过期

检查：

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com
```

### 私钥权限错误

Nginx reload 失败，error log 或 `nginx -t` 中可能提示无法读取 key。

检查：

```bash
sudo nginx -t
ls -l /etc/letsencrypt/live/example.com/
```

### 80 端口不可访问导致签发失败

HTTP-01 验证需要公网能访问 80 端口。检查云安全组、防火墙、Nginx server 配置。

## 五、TLS 配置保守模板

很多发行版或 Certbot 会自动生成合理配置。学习阶段不建议你盲目复制复杂密码套件。

保守写法：

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

注意：如果你的 Nginx 版本较新，`http2` 的写法可能有更新形式。以当前服务器 `nginx -t` 和官方文档为准。

## 六、生产上线检查清单

- 域名 A 记录或 CNAME 指向正确服务器。
- 80 和 443 端口在云安全组、防火墙中开放。
- `server_name` 与证书域名一致。
- HTTP 能跳转 HTTPS。
- `certbot renew --dry-run` 成功。
- Nginx reload 成功。
- Go 服务没有直接暴露公网。
- `X-Forwarded-Proto` 设置正确。

## 七、本节练习

1. 查看 Certbot 自动续期 timer。
2. 执行 `certbot renew --dry-run`。
3. 用 openssl 检查证书有效期。
4. 故意访问错误域名，观察证书不匹配错误。
5. 写一份你自己的 HTTPS 上线检查清单。

## 八、你应该掌握

学完本节，你应该不仅会“配 HTTPS”，还知道证书如何续期、如何验证、为什么线上证书过期是严重事故。

