# 1. upstream 与三实例 Go 服务

本节目标：启动三个 Go 服务实例，并使用 Nginx `upstream` 把请求分发到它们。

---

## 一、启动三个 Go 实例

使用第 4 阶段的 Go 服务。

终端 1：

```bash
PORT=8080 go run main.go
```

终端 2：

```bash
PORT=8081 go run main.go
```

终端 3：

```bash
PORT=8082 go run main.go
```

PowerShell：

```powershell
$env:PORT="8080"; go run main.go
```

分别验证：

```bash
curl http://127.0.0.1:8080/api/ping
curl http://127.0.0.1:8081/api/ping
curl http://127.0.0.1:8082/api/ping
```

应该看到不同的 `port`。

---

## 二、配置 upstream

`upstream` 写在 `http` 上下文中。放在 `conf.d` 文件里时，通常也是被 include 到 `http` 里。

```nginx
upstream go_api {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://go_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

检查并重载：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 三、连续请求观察分发

```bash
for i in {1..12}; do curl -s -H "Host: api.local" http://127.0.0.1/api/ping; echo; done
```

你应该看到端口在 `8080`、`8081`、`8082` 之间变化。

这就是默认轮询。

---

## 四、upstream 名称是什么

```nginx
upstream go_api {
}
```

这里的 `go_api` 是 Nginx 内部名称，不是 DNS 域名。

使用时：

```nginx
proxy_pass http://go_api;
```

Nginx 会把它解析到 upstream 中配置的 server 列表。

---

## 五、查看实际 upstream

可以在日志中加入：

```nginx
log_format api_main '$remote_addr "$request" $status upstream=$upstream_addr';
```

然后：

```nginx
access_log /var/log/nginx/api_access.log api_main;
```

请求后查看：

```bash
sudo tail -f /var/log/nginx/api_access.log
```

你能看到请求实际去了哪个后端地址。

---

## 六、常见问题

### 1. Nginx 返回 502

检查三个 Go 实例是否都启动：

```bash
curl http://127.0.0.1:8080/api/ping
curl http://127.0.0.1:8081/api/ping
curl http://127.0.0.1:8082/api/ping
```

看错误日志：

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

### 2. 只看到一个端口

可能原因：

- 你只启动了一个实例。
- Go 返回内容没有包含端口。
- 配置没有 reload。
- 请求被缓存。

---

## 七、本节练习

1. 启动三个 Go 实例。
2. 配置 `upstream go_api`。
3. 连续请求 12 次。
4. 在日志中加入 `$upstream_addr`。
5. 停掉 `8081`，再次请求并观察错误日志。

---

## 八、本节复盘

请确认你能回答：

1. `upstream` 应该写在哪个上下文？
2. `proxy_pass http://go_api` 中的 `go_api` 是什么？
3. 默认负载均衡策略是什么？
4. 如何知道请求实际去了哪个 Go 实例？
5. 停掉一个实例后为什么可能出现 502？

