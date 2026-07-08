# 03 准备阶段：一次 HTTP 请求的完整链路

## 本节目标

把浏览器、DNS、TCP、TLS、Nginx、Go 服务串成一条完整链路。很多 Nginx 问题本质上是你不知道请求卡在哪一段。

## 一、普通 HTTP 请求链路

以访问 `http://api.local/api/ping` 为例：

```text
浏览器或 curl
  |
  | 1. 查询 api.local 对应的 IP
  v
DNS 或 hosts
  |
  | 2. 得到 127.0.0.1
  v
TCP 连接到 127.0.0.1:80
  |
  | 3. 发送 HTTP 请求
  v
Nginx
  |
  | 4. 根据 Host 匹配 server_name
  | 5. 根据 URI 匹配 location
  | 6. proxy_pass 转发给 Go
  v
Go API :8080
  |
  | 7. Go handler 处理请求
  v
Nginx
  |
  | 8. 记录 access.log
  v
客户端收到响应
```

## 二、HTTPS 请求多了什么

访问 `https://api.local/api/ping` 时，客户端和 Nginx 之间多了 TLS：

```text
Client -> TCP 443 -> TLS 握手 -> HTTP over TLS -> Nginx -> Go
```

如果 Nginx 做 TLS 终止，那么 Go 服务通常仍然收到普通 HTTP 请求：

```text
Client --HTTPS--> Nginx --HTTP--> Go
```

所以 Go 服务想知道用户原本访问的是 HTTPS，需要依赖：

```nginx
proxy_set_header X-Forwarded-Proto https;
```

## 三、Nginx 如何选择 server

Nginx 会根据：

- 监听端口：`listen 80` 或 `listen 443 ssl`
- 请求 Host：例如 `api.local`
- `server_name`

选择 server。

示例：

```nginx
server {
    listen 80;
    server_name api.local;
}

server {
    listen 80;
    server_name static.local;
}
```

测试时即使没有配置 hosts，也可以用：

```bash
curl -H "Host: api.local" http://127.0.0.1/api/ping
```

## 四、Nginx 如何选择 location

进入某个 `server` 后，Nginx 再根据 URI 选择 `location`。

例如请求：

```text
GET /api/ping
```

会匹配：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

如果没有更具体的匹配，可能落到：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

## 五、每一层的常见问题

### DNS 或 hosts

现象：

```text
Could not resolve host
```

检查：

```bash
ping api.local
cat /etc/hosts
```

### TCP 连接

现象：

```text
Connection refused
```

检查：

```bash
ss -lntp | grep ':80'
systemctl status nginx
```

### TLS 证书

现象：

```text
certificate verify failed
```

学习环境可用：

```bash
curl -k https://api.local
```

生产环境应检查证书是否有效、域名是否匹配、是否过期。

### Nginx 到 Go

现象：

```text
502 Bad Gateway
```

检查：

```bash
curl http://127.0.0.1:8080/api/ping
tail -f /var/log/nginx/error.log
```

### Go 服务慢

现象：

```text
504 Gateway Timeout
```

检查：

```bash
time curl http://127.0.0.1:8080/api/slow
tail -f /var/log/nginx/access.log
```

## 六、练习：画出自己的链路

请你画出下面请求的链路：

```text
https://app.local/api/users?id=1
```

至少标出：

- DNS 或 hosts。
- TCP 端口。
- TLS 是否发生。
- 匹配哪个 server。
- 匹配哪个 location。
- 转发到哪个 Go 端口。
- 日志记录在哪里。

## 七、你应该掌握

学完本节，你应该能把一个请求从客户端到 Go handler 的路径讲清楚。以后排障时，你会先定位层级，再查具体配置。

