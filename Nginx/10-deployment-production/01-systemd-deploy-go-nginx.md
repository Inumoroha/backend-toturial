# 01 生产部署：systemd 管理 Go 服务，Nginx 对外代理

## 本节目标

学习最经典的单机部署方式：

```text
systemd 管理 Go 服务
Nginx 监听 80/443
Nginx 反向代理到本机 Go 端口
```

这套方式适合小型项目、个人项目、教学环境和很多传统服务器部署场景。

## 一、构建 Go 程序

在 Go 项目目录执行：

```bash
go build -o myapp main.go
```

建议部署到：

```bash
sudo mkdir -p /opt/myapp
sudo cp myapp /opt/myapp/myapp
```

创建配置目录：

```bash
sudo mkdir -p /etc/myapp
```

## 二、创建运行用户

不要用 root 跑业务服务。

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp
sudo chown -R myapp:myapp /opt/myapp
```

## 三、编写 systemd service

创建 `/etc/systemd/system/myapp.service`：

```ini
[Unit]
Description=My Go API Service
After=network.target

[Service]
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/myapp
Environment=PORT=8080
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

加载并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl start myapp
sudo systemctl enable myapp
sudo systemctl status myapp
```

查看日志：

```bash
journalctl -u myapp -f
```

## 四、Nginx 代理 systemd 服务

创建 `/etc/nginx/conf.d/myapp.conf`：

```nginx
server {
    listen 80;
    server_name api.example.com;

    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log warn;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 五、发布新版本

基本流程：

```bash
go build -o myapp main.go
sudo systemctl stop myapp
sudo cp myapp /opt/myapp/myapp
sudo chown myapp:myapp /opt/myapp/myapp
sudo systemctl start myapp
sudo systemctl status myapp
```

更稳妥的做法是保留历史版本：

```text
/opt/myapp/releases/20260705-1800/myapp
/opt/myapp/current -> /opt/myapp/releases/20260705-1800
```

systemd 中运行：

```ini
ExecStart=/opt/myapp/current/myapp
```

回滚时只需要切换 `current` 软链接并重启。

## 六、部署检查表

发布后检查：

```bash
sudo systemctl status myapp
curl http://127.0.0.1:8080/api/ping
sudo nginx -t
curl http://api.example.com/api/ping
sudo tail -n 20 /var/log/nginx/myapp_access.log
sudo tail -n 20 /var/log/nginx/myapp_error.log
```

## 七、本节练习

1. 构建一个 Go API 二进制文件。
2. 创建 systemd service。
3. 用 systemd 启动 Go 服务。
4. 配置 Nginx 代理到本机端口。
5. 模拟发布新版本和回滚。

## 八、你应该掌握

学完本节，你应该能把一个 Go API 服务部署到 Linux 服务器，并由 Nginx 对外提供访问入口。

