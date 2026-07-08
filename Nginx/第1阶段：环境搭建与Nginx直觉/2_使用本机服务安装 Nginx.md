# 2. 使用本机服务安装 Nginx

本节目标：在 Linux 或 WSL 环境中安装 Nginx，并使用 systemd 管理它。生产环境中，你经常会遇到这种方式。

上一节使用 Docker 是为了快速体验。这一节开始接近真实服务器环境：

```text
Nginx 作为系统服务运行。
配置文件放在 /etc/nginx。
日志文件放在 /var/log/nginx。
systemd 负责启动、停止、重启、开机自启。
```

---

## 一、什么时候使用系统安装

系统安装适合：

- 云服务器部署。
- 传统虚拟机部署。
- 不使用容器的项目。
- 学习 systemd、日志和 Linux 服务管理。

Docker 适合：

- 本地练习。
- 容器化项目。
- 快速重建环境。

两种方式都要会。Go 后端工程师线上会遇到各种部署形态。

---

## 二、Ubuntu / Debian 安装

更新软件源：

```bash
sudo apt update
```

安装：

```bash
sudo apt install -y nginx
```

查看版本：

```bash
nginx -v
```

启动：

```bash
sudo systemctl start nginx
```

设置开机自启：

```bash
sudo systemctl enable nginx
```

查看状态：

```bash
sudo systemctl status nginx
```

验证：

```bash
curl http://127.0.0.1
```

---

## 三、CentOS / Rocky Linux / AlmaLinux 安装

安装：

```bash
sudo dnf install -y nginx
```

启动：

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

查看状态：

```bash
sudo systemctl status nginx
```

如果访问不了，可能需要开放防火墙：

```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```

---

## 四、systemd 常用命令

启动：

```bash
sudo systemctl start nginx
```

停止：

```bash
sudo systemctl stop nginx
```

重启：

```bash
sudo systemctl restart nginx
```

平滑重载：

```bash
sudo systemctl reload nginx
```

查看状态：

```bash
sudo systemctl status nginx
```

查看日志：

```bash
journalctl -u nginx -f
```

注意：

```text
restart 会重启进程。
reload 会让 Nginx 重新加载配置，通常更适合配置变更。
```

---

## 五、配置文件在哪里

常见路径：

```text
/etc/nginx/nginx.conf
/etc/nginx/conf.d/*.conf
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
```

Ubuntu 系有时会使用：

```text
/etc/nginx/sites-available/default
/etc/nginx/sites-enabled/default
```

查看主配置：

```bash
sudo less /etc/nginx/nginx.conf
```

查看完整配置：

```bash
sudo nginx -T
```

`nginx -T` 非常重要。它会把主配置和所有 `include` 进来的配置都打印出来。

---

## 六、日志文件在哪里

常见路径：

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

访问日志：

```bash
sudo tail -f /var/log/nginx/access.log
```

错误日志：

```bash
sudo tail -f /var/log/nginx/error.log
```

打开一个终端看访问日志：

```bash
sudo tail -f /var/log/nginx/access.log
```

另一个终端请求：

```bash
curl http://127.0.0.1
```

你应该能看到日志新增一行。

---

## 七、配置检查

修改任何 Nginx 配置后，先执行：

```bash
sudo nginx -t
```

成功输出类似：

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

然后 reload：

```bash
sudo systemctl reload nginx
```

不要改完配置直接 restart。学习阶段要养成：

```text
修改配置 -> nginx -t -> reload -> curl 验证 -> 看日志
```

---

## 八、端口被占用怎么处理

如果启动失败：

```text
bind() to 0.0.0.0:80 failed
```

说明 80 端口可能被占用。

查看监听端口：

```bash
sudo ss -lntp | grep ':80'
```

或：

```bash
sudo lsof -i :80
```

如果你之前启动了 Docker Nginx，并映射到 80，也可能造成冲突。

---

## 九、本节练习

1. 在 Linux 或 WSL 中安装 Nginx。
2. 启动 Nginx 并设置开机自启。
3. 使用 `curl http://127.0.0.1` 验证。
4. 找到 `/etc/nginx/nginx.conf`。
5. 使用 `nginx -T` 查看完整配置。
6. 查看 access log 和 error log。
7. 停止 Nginx，再次访问，观察错误。
8. 启动 Nginx，确认恢复。

---

## 十、本节复盘

请确认你能回答：

1. `restart` 和 `reload` 有什么区别？
2. Nginx 主配置文件通常在哪里？
3. access log 和 error log 分别记录什么？
4. `nginx -t` 能检查什么，不能检查什么？
5. 80 端口被占用时如何排查？

