# 03 深入理解：Nginx 请求处理阶段

## 本节目标

从整体上理解 Nginx 收到请求后经历哪些处理阶段。你不需要背源码，但要知道配置为什么会按某种顺序生效。

## 一、简化处理流程

```text
接收连接
  |
读取请求行和请求头
  |
选择 server
  |
选择 location
  |
rewrite 阶段
  |
访问控制阶段
  |
内容处理阶段
  |
日志阶段
```

## 二、server 选择阶段

Nginx 根据监听地址、端口和 Host 选择 server。

```nginx
server {
    listen 80;
    server_name api.local;
}
```

如果没有匹配到 `server_name`，会使用该监听端口的默认 server。

可以显式声明默认 server：

```nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;
}
```

`444` 是 Nginx 特有状态，表示直接关闭连接。生产中有些人用它丢弃未知 Host 请求。

## 三、rewrite 阶段

例如：

```nginx
rewrite ^/old/(.*)$ /new/$1 last;
```

`last` 会让 Nginx 用新的 URI 重新选择 location。

如果 rewrite 规则太多，请求路径会变得难以推理。优先使用清晰的 location 和 `return`。

## 四、访问控制阶段

这些通常发生在内容处理之前：

```nginx
allow 10.0.0.0/8;
deny all;

auth_basic "Admin";
auth_basic_user_file /etc/nginx/.htpasswd;

limit_req zone=api_limit burst=20 nodelay;
```

如果访问控制拒绝，请求可能不会到达 Go 服务。

这就是为什么 access log 中可能有状态码，但 Go 服务没有任何日志。

## 五、内容处理阶段

常见内容处理方式：

### Nginx 直接返回

```nginx
return 200 "ok\n";
```

### 静态文件

```nginx
root /var/www/app;
try_files $uri $uri/ /index.html;
```

### 反向代理

```nginx
proxy_pass http://127.0.0.1:8080;
```

一个 location 中通常应该只有一种主要内容处理方式，避免让配置难以理解。

## 六、日志阶段

请求处理完成后，Nginx 记录 access log。

如果客户端提前断开，可能看到：

```text
499
```

`499` 是 Nginx 记录的状态，表示客户端关闭连接。比如用户取消请求、浏览器超时、客户端主动断开。

## 七、为什么 Go 没收到请求

如果 Go 没日志，但 Nginx access log 有记录，可能是：

- Nginx 直接 `return` 了。
- 命中了静态文件 location。
- 被 `allow/deny` 拒绝。
- 被 `limit_req` 限流。
- `try_files` 走了其他路径。
- `location` 匹配不是你以为的那个。

排查：

```bash
sudo nginx -T
curl -v http://api.local/path
tail -f /var/log/nginx/access.log
```

## 八、本节练习

1. 配置一个默认 server 返回 444。
2. 配置 rewrite，并观察 location 重新匹配。
3. 配置限流，让请求不进入 Go。
4. 配置静态 location 和 API location，观察不同请求的处理方式。
5. 解释一次请求完整经历了哪些阶段。

## 九、你应该掌握

学完本节，你应该能解释为什么某些请求没有进入 Go 服务，以及 Nginx 配置大致按什么阶段发挥作用。

