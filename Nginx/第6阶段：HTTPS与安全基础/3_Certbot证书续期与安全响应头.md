# 3. Certbot 证书续期与安全响应头

本节目标：了解生产环境如何使用 Let's Encrypt / Certbot 管理证书，并掌握常见安全响应头。

---

## 一、使用 Certbot 申请证书

Ubuntu / Debian：

```bash
sudo apt install -y certbot python3-certbot-nginx
```

申请：

```bash
sudo certbot --nginx -d api.example.com
```

证书常见路径：

```text
/etc/letsencrypt/live/api.example.com/fullchain.pem
/etc/letsencrypt/live/api.example.com/privkey.pem
```

---

## 二、证书续期

查看自动续期：

```bash
systemctl list-timers | grep certbot
```

演练续期：

```bash
sudo certbot renew --dry-run
```

`--dry-run` 不会真正替换证书，适合测试续期流程。

---

## 三、续期后重载 Nginx

证书续期后，Nginx 需要 reload 才会加载新证书。

可以使用：

```bash
sudo certbot renew --deploy-hook "systemctl reload nginx"
```

---

## 四、检查证书有效期

本地文件：

```bash
openssl x509 -in /etc/letsencrypt/live/api.example.com/fullchain.pem -noout -dates
```

线上域名：

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
```

`-servername` 用于 SNI，多域名同 IP 时很重要。

---

## 五、常见安全响应头

```nginx
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy no-referrer-when-downgrade always;
```

如果确认 HTTPS 长期可用，可以考虑 HSTS：

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

注意：HSTS 不要在不稳定的 HTTPS 环境里随便开很长时间。

---

## 六、Basic Auth

适合临时保护后台或测试环境：

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

配置：

```nginx
location /admin/ {
    auth_basic "Admin";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:8080;
}
```

---

## 七、本节练习

1. 查看 Certbot timer。
2. 执行 `certbot renew --dry-run`。
3. 用 openssl 查看证书有效期。
4. 添加基础安全响应头。
5. 给 `/admin/` 配置 Basic Auth。

---

## 八、本节复盘

请确认你能回答：

1. Let's Encrypt 证书为什么需要自动续期？
2. `certbot renew --dry-run` 有什么作用？
3. 续期后为什么要 reload Nginx？
4. HSTS 为什么要谨慎开启？
5. Basic Auth 适合什么场景？

