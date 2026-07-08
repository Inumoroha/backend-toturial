# 06：Nginx 部署项目

本节目标：安装 Nginx，修改默认首页，启动服务，设置开机自启，验证 HTTP 响应，并查看日志。

本节命令顺序：

```text
1. apt install nginx
2. systemctl status nginx
3. systemctl start nginx
4. systemctl enable nginx
5. curl -I localhost
6. tee 写入网页
7. journalctl -u nginx
8. tail Nginx 日志
```

说明：如果 WSL 不支持 systemd，请在虚拟机或测试服务器中完成。

---

## 1. 安装 Nginx

```bash
sudo apt update
sudo apt install nginx
```

查看版本：

```bash
nginx -v
```

查看服务：

```bash
systemctl status nginx
```

---

## 2. 启动并设置自启

启动：

```bash
sudo systemctl start nginx
```

确认运行：

```bash
systemctl is-active nginx
```

设置开机自启：

```bash
sudo systemctl enable nginx
```

确认：

```bash
systemctl is-enabled nginx
```

---

## 3. 验证 HTTP 响应

查看响应头：

```bash
curl -I http://localhost
```

只看状态码：

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost
```

查看端口：

```bash
ss -tuln | grep ':80'
```

查看进程：

```bash
ps aux | grep nginx | grep -v grep
```

---

## 4. 修改默认首页

备份默认首页：

```bash
sudo cp /var/www/html/index.nginx-debian.html /var/www/html/index.nginx-debian.html.bak 2>/dev/null || true
```

写入新首页：

```bash
cat <<'EOF' | sudo tee /var/www/html/index.html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Linux Stage 08</title>
  </head>
  <body>
    <h1>Linux Stage 08</h1>
    <p>Nginx deployment completed.</p>
  </body>
</html>
EOF
```

验证：

```bash
curl http://localhost
```

---

## 5. 查看 Nginx 日志

systemd 日志：

```bash
journalctl -u nginx -n 50 --no-pager
```

最近 1 小时：

```bash
journalctl -u nginx --since "1 hour ago" --no-pager
```

Nginx 访问日志：

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

Nginx 错误日志：

```bash
sudo tail -n 20 /var/log/nginx/error.log
```

---

## 6. 重启和重载

测试配置：

```bash
sudo nginx -t
```

重载配置：

```bash
sudo systemctl reload nginx
```

重启服务：

```bash
sudo systemctl restart nginx
```

再次验证：

```bash
curl -I http://localhost
```

---

## 7. 写入部署记录

进入报告目录：

```bash
cd ~/linux-practice/stage-08
```

写入记录：

```bash
{
  echo "# Nginx Deployment Record"
  echo
  date
  echo
  echo "## Service"
  systemctl is-active nginx
  systemctl is-enabled nginx
  echo
  echo "## HTTP"
  curl -I http://localhost
} > reports/nginx-deployment.md
```

查看：

```bash
cat reports/nginx-deployment.md
```

---

## 8. 本节小结

必须掌握：

```bash
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
systemctl status nginx
curl -I http://localhost
ss -tuln | grep ':80'
sudo nginx -t
sudo systemctl reload nginx
journalctl -u nginx -n 50 --no-pager
sudo tail -n 20 /var/log/nginx/access.log
```

完成后进入下一节：`07-troubleshooting-labs.md`。

