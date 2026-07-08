# Nginx 面试题与参考答案

这份文档用于准备 Go 后端工程师面试中可能遇到的 Nginx 相关问题。

它不是为了让你死记硬背，而是帮助你做到：

```text
听得懂面试官在问什么。
知道该从哪个角度回答。
能结合 Go 后端项目解释 Nginx 的作用。
遇到 502、504、HTTPS、反向代理、负载均衡等追问不慌。
```

建议复习方式：

1. 先看题目，自己尝试回答。
2. 再看参考答案。
3. 最后把答案改成自己的表达。
4. 尽量结合自己的 Go API、Nginx 网关项目、静态资源部署、HTTPS 和日志排障经验讲。

---

## 一、Nginx 基础概念类

### 1. Nginx 是什么？

参考回答：

Nginx 是一个高性能的 Web 服务器，也常用作反向代理服务器、负载均衡器和静态资源服务器。

在 Go 后端项目中，Nginx 通常放在 Go 服务前面：

```text
Client -> Nginx -> Go API
```

它可以负责：

- 对外监听 80、443 端口。
- 提供 HTTPS。
- 转发请求到 Go 服务。
- 托管静态资源。
- 做负载均衡。
- 记录访问日志。
- 做基础限流和安全控制。

追问点：

- Nginx 和 Go 服务分别负责什么？
- Nginx 为什么适合做入口层？

可以补充：

我理解 Nginx 更适合做通用入口能力，比如代理、HTTPS、静态资源、日志、限流；Go 服务更适合做业务逻辑，比如鉴权、订单、支付、数据一致性。

---

### 2. Nginx 常见使用场景有哪些？

参考回答：

常见场景包括：

- 静态资源服务。
- 反向代理后端服务。
- 负载均衡。
- HTTPS TLS 终止。
- HTTP 跳转 HTTPS。
- API 网关的基础能力。
- 请求限流。
- 代理缓存。
- 日志记录和故障排查。

在前后端分离项目中，常见配置是：

```text
/              -> 前端静态页面
/assets/       -> 静态资源缓存
/api/          -> 反向代理到 Go API
```

---

### 3. Nginx 和 Apache 有什么区别？

参考回答：

Nginx 和 Apache 都可以作为 Web 服务器。Nginx 使用事件驱动、异步非阻塞模型，处理高并发连接时资源占用较低。Apache 传统上更多使用进程或线程模型，功能模块丰富，历史更久。

在现代后端项目中，Nginx 常用于入口层、反向代理、静态资源和负载均衡。

面试中不需要贬低 Apache，可以说：

```text
两者都成熟。Nginx 在高并发连接、反向代理和静态资源场景中使用非常广泛，我在 Go 后端项目中更常接触 Nginx。
```

---

### 4. 正向代理和反向代理有什么区别？

参考回答：

正向代理代理的是客户端。客户端知道自己在使用代理，例如访问外网时通过代理服务器转发请求。

反向代理代理的是服务端。客户端访问的是 Nginx，但实际服务可能在后面的 Go、Java、Node 等服务上。

示例：

```text
正向代理：Client -> Proxy -> Internet
反向代理：Client -> Nginx -> Go API
```

Nginx 在后端项目里最常用的是反向代理。

---

### 5. Nginx 为什么性能高？

参考回答：

Nginx 使用 master/worker 多进程模型和事件驱动、非阻塞 I/O。一个 worker 可以处理大量连接，而不是每个连接都创建一个线程。

它适合处理大量并发连接、静态文件传输和代理转发。

可以从几个方面回答：

- master 管理 worker。
- worker 处理连接。
- 事件驱动减少线程切换开销。
- 非阻塞 I/O 适合高并发网络请求。
- sendfile 等机制提升静态文件传输效率。

---

### 6. Nginx 的 master 和 worker 分别做什么？

参考回答：

master 进程负责：

- 读取和校验配置。
- 管理 worker 进程。
- 接收 reload、stop 等信号。
- 平滑重载配置。

worker 进程负责：

- 接收客户端连接。
- 读取请求。
- 匹配 server 和 location。
- 返回静态文件或代理到 upstream。
- 写访问日志。

可以用命令查看：

```bash
ps -ef | grep nginx
```

---

### 7. Nginx reload 和 restart 有什么区别？

参考回答：

`reload` 是平滑重载配置。Nginx master 会加载新配置，启动新的 worker，旧 worker 处理完已有请求后退出。

`restart` 是重启服务，通常会停止再启动，影响更大。

修改配置后的推荐流程：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

先检查配置，再 reload。

---

### 8. `nginx -t` 和 `nginx -T` 有什么区别？

参考回答：

`nginx -t` 用于检查配置语法是否正确。

`nginx -T` 会检查配置，并打印最终加载的完整配置，包括 `include` 引入的文件。

排查配置是否生效时，`nginx -T` 很有用：

```bash
sudo nginx -T | grep api.local
```

注意：`nginx -t` 成功只代表语法正确，不代表后端端口、文件路径、业务逻辑一定正确。

---

## 二、配置结构与请求匹配类

### 9. Nginx 配置文件有哪些常见上下文？

参考回答：

常见上下文包括：

```text
main
events
http
server
location
```

示例：

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name api.local;

        location / {
            return 200 "ok\n";
        }
    }
}
```

常见位置：

- `upstream` 写在 `http`。
- `server` 写在 `http`。
- `location` 写在 `server`。
- `proxy_pass` 常写在 `location`。

---

### 10. 如果 Nginx 报 `"server" directive is not allowed here`，是什么意思？

参考回答：

说明 `server` 指令写错了上下文。

错误示例：

```nginx
events {
    server {
        listen 80;
    }
}
```

`server` 应该写在 `http` 上下文中。

排查时看：

```bash
sudo nginx -t
sudo nginx -T
```

---

### 11. Nginx 的 `include` 有什么作用？

参考回答：

`include` 用于把多个配置文件组合进主配置。

例如：

```nginx
include /etc/nginx/conf.d/*.conf;
```

表示加载 `conf.d` 目录下所有 `.conf` 文件。

生产中通常会拆分配置：

```text
00-log-format.conf
10-upstream.conf
20-api.conf
30-static.conf
```

这样职责更清楚，也更容易维护和回滚。

---

### 12. Nginx 如何根据域名匹配 server？

参考回答：

Nginx 会根据监听端口和请求头中的 `Host` 匹配 `server_name`。

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

测试时可以用：

```bash
curl -H "Host: api.local" http://127.0.0.1
```

如果没有匹配到，会进入默认 server。

---

### 13. `default_server` 是什么？

参考回答：

`default_server` 指定某个 listen 端口上的默认 server。当请求的 Host 没有匹配任何 `server_name` 时，会进入默认 server。

示例：

```nginx
server {
    listen 80 default_server;
    server_name _;
    return 404 "unknown host\n";
}
```

生产中可以用默认 server 拒绝未知 Host 请求，减少 Host Header 攻击面。

---

### 14. location 有哪些匹配方式？

参考回答：

常见方式：

```nginx
location = /health {}
location ^~ /assets/ {}
location /api/ {}
location ~ \.txt$ {}
location / {}
```

含义：

- `=` 精确匹配。
- `^~` 前缀匹配，命中后不再检查正则。
- 普通前缀匹配。
- `~` 正则匹配，区分大小写。
- `/` 兜底匹配。

---

### 15. location 匹配优先级大致是什么？

参考回答：

简化理解：

1. 先找 `=` 精确匹配。
2. 再找最长前缀匹配。
3. 如果命中 `^~`，停止检查正则。
4. 否则可能继续检查正则。
5. 最后使用 `/` 兜底。

面试中可以补充：

```text
实际规则有细节，但工程上我会尽量避免写过于复杂的 location，优先保证配置清晰可维护。
```

---

### 16. `$uri` 和 `$request_uri` 有什么区别？

参考回答：

请求：

```text
/api/users?id=1
```

通常：

```text
$uri = /api/users
$request_uri = /api/users?id=1
```

`$uri` 是规范化后的 URI，不包含查询参数。

`$request_uri` 是原始请求 URI，包含查询参数。

HTTP 跳 HTTPS 时通常用：

```nginx
return 301 https://$host$request_uri;
```

这样不会丢失路径和查询参数。

---

### 17. `return` 和 `rewrite` 有什么区别？

参考回答：

`return` 通常用于直接返回响应或重定向，简单清晰：

```nginx
return 301 https://$host$request_uri;
```

`rewrite` 用于 URI 改写，适合更复杂的路径转换：

```nginx
rewrite ^/old/(.*)$ /new/$1 last;
```

工程上，如果只是跳转，优先用 `return`。复杂路径改写再考虑 `rewrite`。

---

## 三、静态资源服务类

### 18. Nginx 如何配置静态资源服务？

参考回答：

示例：

```nginx
server {
    listen 80;
    server_name static.local;

    root /var/www/app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Nginx 会根据请求 URI 去磁盘上找对应文件。

---

### 19. `root` 和 `alias` 有什么区别？

参考回答：

`root` 的路径规则：

```text
真实路径 = root 路径 + 完整 URI
```

示例：

```nginx
location /assets/ {
    root /var/www/app;
}
```

请求 `/assets/style.css`，真实路径是：

```text
/var/www/app/assets/style.css
```

`alias` 的路径规则：

```text
真实路径 = alias 路径 + 去掉 location 前缀后的 URI
```

示例：

```nginx
location /downloads/ {
    alias /data/files/;
}
```

请求 `/downloads/a.pdf`，真实路径是：

```text
/data/files/a.pdf
```

---

### 20. `try_files` 有什么作用？

参考回答：

`try_files` 按顺序尝试文件是否存在。

```nginx
try_files $uri $uri/ =404;
```

表示：

1. 先找 `$uri` 对应的文件。
2. 再找 `$uri/` 对应的目录。
3. 都没有就返回 404。

前端 SPA 常用：

```nginx
try_files $uri $uri/ /index.html;
```

这样刷新 `/users/1` 时会返回 `index.html`，交给前端路由处理。

---

### 21. 为什么前端 SPA 刷新页面会 404？怎么解决？

参考回答：

SPA 中 `/users/1` 是前端路由，不一定是磁盘上的真实文件。刷新时浏览器直接请求 `/users/1`，Nginx 去磁盘找这个路径，找不到就 404。

解决：

```nginx
location / {
    root /var/www/app;
    try_files $uri $uri/ /index.html;
}
```

同时 API 要单独写：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

避免 API 被 `index.html` 吃掉。

---

### 22. 静态资源如何配置缓存？

参考回答：

对于带 hash 的静态资源：

```nginx
location ^~ /assets/ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

对于 `index.html`：

```nginx
location = /index.html {
    add_header Cache-Control "no-cache";
}
```

原因是 JS/CSS 带 hash 可以长缓存，HTML 需要及时更新，否则可能引用旧资源。

---

### 23. Nginx 如何开启 gzip？

参考回答：

在 `http` 上下文配置：

```nginx
gzip on;
gzip_min_length 1024;
gzip_comp_level 5;
gzip_types text/plain text/css application/json application/javascript;
```

gzip 适合文本类资源，比如 HTML、CSS、JS、JSON。不适合已经压缩过的图片和视频。

验证：

```bash
curl -H "Accept-Encoding: gzip" -I http://static.local/assets/app.js
```

---

### 24. 静态资源 403 和 404 如何排查？

参考回答：

404 主要查路径：

- `root` / `alias` 是否写对。
- 文件是否存在。
- 请求是否匹配到正确 location。
- `try_files` 是否按预期执行。

403 主要查权限：

- Nginx worker 用户是否能读文件。
- 上级目录是否有执行权限。
- 是否访问目录但没有 index。
- 是否配置了 `deny all`。

常用命令：

```bash
sudo nginx -T
namei -l /var/www/app/index.html
sudo tail -n 50 /var/log/nginx/error.log
```

---

## 四、反向代理 Go 服务类

### 25. Nginx 如何反向代理 Go 服务？

参考回答：

Go 服务监听本地端口：

```text
127.0.0.1:8080
```

Nginx 配置：

```nginx
server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

请求链路：

```text
Client -> Nginx -> Go API
```

---

### 26. `proxy_pass` 后面带 `/` 和不带 `/` 有什么区别？

参考回答：

示例一：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

请求 `/api/ping`，Go 收到 `/api/ping`。

示例二：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

请求 `/api/ping`，Go 收到 `/ping`。

因为 `proxy_pass` 带 URI 时，会用后面的 URI 替换 location 匹配的前缀。

面试中可以说：

```text
这是 Nginx 新手非常容易踩的坑。如果 Nginx 访问 404 但 Go 直连正常，我会优先检查 proxy_pass 路径规则。
```

---

### 27. 常见代理头有哪些？

参考回答：

常见代理头：

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

含义：

- `Host`：保留原始域名。
- `X-Real-IP`：Nginx 看到的客户端 IP。
- `X-Forwarded-For`：代理链路 IP 列表。
- `X-Forwarded-Proto`：原始请求协议。

Go 服务可以用这些头记录日志、生成回调 URL、判断 HTTPS。

---

### 28. Go 服务为什么拿不到真实客户端 IP？

参考回答：

因为 Go 服务直接连接的是 Nginx。Go 中的 `r.RemoteAddr` 通常是 Nginx 到 Go 的连接地址，比如 `127.0.0.1:xxxxx`。

要获取客户端 IP，需要 Nginx 传：

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Go 再读取这些头。

但要注意，客户端可以伪造 `X-Forwarded-For`，所以真实 IP 的可信性来自网络边界，比如 Go 服务不能被外部绕过 Nginx 直接访问。

---

### 29. `X-Forwarded-For` 为什么不能无条件信任？

参考回答：

因为客户端可以自己构造请求头：

```bash
curl -H "X-Forwarded-For: 1.2.3.4" http://api.local
```

如果后端无条件相信第一个 IP，就可能被伪造。

正确思路：

- Go 服务只接受可信 Nginx 的请求。
- 多层代理时使用 `set_real_ip_from` 指定可信代理。
- 不要把 `set_real_ip_from` 随便写成 `0.0.0.0/0`。

---

### 30. Nginx 代理 Go 服务时常见超时配置有哪些？

参考回答：

常见配置：

```nginx
proxy_connect_timeout 5s;
proxy_send_timeout 30s;
proxy_read_timeout 30s;
```

含义：

- `proxy_connect_timeout`：连接后端 Go 服务的超时。
- `proxy_send_timeout`：向后端发送请求的超时。
- `proxy_read_timeout`：等待后端响应的超时。

504 常常和 `proxy_read_timeout` 有关，但不能一看到 504 就调大超时，还要查 Go 服务、数据库和外部依赖。

---

### 31. Nginx 如何限制上传大小？

参考回答：

使用：

```nginx
client_max_body_size 10m;
```

超过限制返回：

```text
413 Request Entity Too Large
```

上传接口可以单独放大：

```nginx
location /api/upload {
    client_max_body_size 100m;
    proxy_pass http://127.0.0.1:8080;
}
```

Go 服务也应该做自己的大小限制，不能只依赖 Nginx。

---

### 32. Nginx 如何代理 WebSocket？

参考回答：

WebSocket 需要协议升级，所以要传 Upgrade 相关头。

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    location /ws/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
    }
}
```

注意 `map` 写在 `http` 上下文。

---

## 五、负载均衡类

### 33. Nginx 如何配置负载均衡？

参考回答：

使用 `upstream`：

```nginx
upstream go_api {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    location /api/ {
        proxy_pass http://go_api;
    }
}
```

默认是轮询，请求会分发到多个 Go 实例。

---

### 34. Nginx 有哪些常见负载均衡策略？

参考回答：

常见策略：

- 默认轮询。
- `weight` 权重。
- `least_conn` 最少连接。
- `ip_hash`。

示例：

```nginx
upstream go_api {
    least_conn;
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}
```

选择：

- 实例能力差不多，用轮询。
- 实例性能不同，用权重。
- 请求耗时差异大，可以考虑最少连接。
- 需要会话粘滞，可以用 `ip_hash`，但不推荐长期依赖。

---

### 35. `weight` 有什么作用？

参考回答：

`weight` 用于设置后端权重。

```nginx
upstream go_api {
    server 127.0.0.1:8080 weight=3;
    server 127.0.0.1:8081 weight=1;
}
```

`8080` 会接收更多请求。

适合后端机器配置不同或实例能力不同的场景。

---

### 36. `least_conn` 适合什么场景？

参考回答：

`least_conn` 会优先把请求分配给当前活跃连接数较少的实例。

适合：

- 请求耗时差异比较大。
- 有慢接口。
- 单纯轮询可能导致某个实例堆积连接。

但是否更好要压测验证，不应该凭感觉切换。

---

### 37. `ip_hash` 有什么作用和风险？

参考回答：

`ip_hash` 会让同一个客户端 IP 尽量落到同一个后端实例，常用于会话粘滞。

风险：

- 用户 IP 可能变化。
- 多个用户可能共用一个出口 IP。
- 扩缩容后映射关系可能变化。
- 容易掩盖服务有状态设计的问题。

更推荐让 Go 服务无状态，比如 session 放 Redis 或使用 JWT。

---

### 38. `max_fails` 和 `fail_timeout` 是什么？

参考回答：

示例：

```nginx
server 127.0.0.1:8080 max_fails=3 fail_timeout=10s;
```

含义：

- `max_fails=3`：在指定时间窗口内失败 3 次后，暂时认为该实例不可用。
- `fail_timeout=10s`：失败统计窗口，也是暂时不可用的时间。

这是被动失败判断，不是完整主动健康检查。

---

### 39. `proxy_next_upstream` 有什么风险？

参考回答：

它用于 upstream 出错时重试其他后端：

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 3;
```

风险在于非幂等请求。

例如 `POST /orders` 第一次请求已经创建订单，但响应断了，Nginx 重试到另一个实例，可能重复创建订单。

所以写接口要考虑幂等，关键写操作不能只依赖 Nginx 重试保证可靠性。

---

### 40. 多实例部署为什么要求 Go 服务尽量无状态？

参考回答：

因为负载均衡下，同一个用户的不同请求可能落到不同实例。

如果 session 存在某个 Go 进程内存中：

```text
登录 -> 8080
下一次请求 -> 8081
8081 找不到 session
```

推荐：

- JWT。
- Redis session。
- 数据库存储。
- 对象存储保存文件。

这样实例可以随时扩缩容和发布。

---

## 六、HTTPS 与安全类

### 41. Nginx 如何配置 HTTPS？

参考回答：

最小配置：

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

证书和私钥路径要正确，私钥不能泄露。

---

### 42. 什么是 TLS 终止？

参考回答：

TLS 终止指 HTTPS 连接在 Nginx 处解密，Nginx 再用 HTTP 转发给后端 Go 服务。

链路：

```text
Client --HTTPS--> Nginx --HTTP--> Go
```

好处：

- 证书集中管理。
- Go 服务不用处理 HTTPS。
- 多个后端服务共享入口层。

此时 Go 如果要知道用户原始协议，需要看 `X-Forwarded-Proto`。

---

### 43. HTTP 如何跳转 HTTPS？

参考回答：

配置一个 80 端口 server：

```nginx
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}
```

`$request_uri` 用于保留路径和查询参数。

---

### 44. `X-Forwarded-Proto` 有什么作用？

参考回答：

当 Nginx 做 TLS 终止时，Go 收到的可能是 HTTP 请求，但用户原始访问是 HTTPS。

Nginx 传：

```nginx
proxy_set_header X-Forwarded-Proto https;
```

Go 可以用它生成正确的外部 URL、OAuth 回调地址、日志字段等。

---

### 45. Let's Encrypt / Certbot 证书如何续期？

参考回答：

Certbot 通常会创建 systemd timer 或 cron 自动续期。

查看：

```bash
systemctl list-timers | grep certbot
```

演练续期：

```bash
sudo certbot renew --dry-run
```

续期后要 reload Nginx：

```bash
sudo certbot renew --deploy-hook "systemctl reload nginx"
```

---

### 46. 常见安全响应头有哪些？

参考回答：

常见配置：

```nginx
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy no-referrer-when-downgrade always;
```

HSTS：

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

HSTS 要谨慎开启，确认 HTTPS 长期稳定后再设置较长时间。

---

### 47. Nginx 如何保护后台路径？

参考回答：

可以使用 IP 白名单：

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    deny all;
    proxy_pass http://127.0.0.1:8080;
}
```

也可以使用 Basic Auth：

```nginx
location /admin/ {
    auth_basic "Admin";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:8080;
}
```

复杂权限仍然应该放在 Go 服务中。

---

## 七、日志与排障类

### 48. access log 和 error log 分别记录什么？

参考回答：

access log 记录每个请求的访问情况，比如 IP、时间、请求路径、状态码、响应大小、耗时、upstream。

error log 记录 Nginx 运行和请求处理中的错误，比如配置错误、文件权限、upstream 连接失败、upstream 超时、请求体过大。

排障通常先看 access log 确认请求是否进入 Nginx，再看 error log 找错误原因。

---

### 49. Nginx 如何记录 upstream 耗时？

参考回答：

自定义日志格式：

```nginx
log_format api_main '$remote_addr "$request" '
                    'status=$status '
                    'rt=$request_time '
                    'uct=$upstream_connect_time '
                    'uht=$upstream_header_time '
                    'urt=$upstream_response_time '
                    'upstream=$upstream_addr';
```

字段：

- `request_time`：整个请求耗时。
- `upstream_connect_time`：连接后端耗时。
- `upstream_header_time`：收到后端响应头耗时。
- `upstream_response_time`：后端响应耗时。
- `upstream_addr`：实际后端实例。

---

### 50. request_id 有什么作用？

参考回答：

request_id 用于把同一次请求在 Nginx 和 Go 日志中关联起来。

Nginx：

```nginx
proxy_set_header X-Request-ID $request_id;
```

日志：

```nginx
log_format api_main 'rid=$request_id ...';
```

Go 日志打印 `X-Request-ID`。

这样排查慢请求或 5xx 时，可以用同一个 ID 找到 Nginx 和 Go 的日志。

---

### 51. 502 Bad Gateway 常见原因和排查步骤？

参考回答：

502 常见原因：

- Go 服务没启动。
- upstream 端口写错。
- Go 服务崩溃。
- Docker 网络地址写错。
- 后端提前断开连接。

排查：

```bash
sudo tail -n 50 /var/log/nginx/error.log
curl http://127.0.0.1:8080/api/ping
ss -lntp | grep 8080
```

典型日志：

```text
connect() failed (111: Connection refused) while connecting to upstream
```

说明 Nginx 连不上后端端口。

---

### 52. 504 Gateway Timeout 常见原因和排查步骤？

参考回答：

504 表示 Nginx 等待 upstream 响应超时。

常见原因：

- Go 接口慢。
- 数据库慢查询。
- 外部 API 慢。
- goroutine 阻塞。
- `proxy_read_timeout` 太短。

排查：

```bash
sudo tail -n 50 /var/log/nginx/error.log
time curl http://127.0.0.1:8080/api/slow
```

典型日志：

```text
upstream timed out while reading response header from upstream
```

不要第一反应就调大超时，要先查为什么 Go 慢。

---

### 53. 499 是什么？

参考回答：

499 是 Nginx 特有状态码，表示客户端主动关闭连接。

常见原因：

- 用户取消请求。
- 浏览器或客户端超时。
- 移动网络断开。
- 后端响应太慢，客户端不等了。
- 上一层代理提前断开。

499 不一定是 Nginx 错，也不一定是 Go 错，要结合请求耗时和客户端超时设置分析。

---

### 54. 413 是什么？怎么处理？

参考回答：

413 表示请求体太大。

Nginx 配置：

```nginx
client_max_body_size 10m;
```

如果上传文件超过限制，就返回 413。

处理方式：

- 给上传接口单独放大限制。
- Go 服务也做文件大小限制。
- 前端提示文件大小限制。

---

### 55. API 返回 HTML 而不是 JSON，可能是什么原因？

参考回答：

常见原因是 API 请求没有进入 `/api/` 代理 location，而是落到了前端 SPA 的兜底：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

排查：

```bash
sudo nginx -T
curl -v http://app.local/api/ping
```

确认是否存在：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

---

## 八、限流、CORS、缓存类

### 56. Nginx 如何做请求限流？

参考回答：

使用 `limit_req_zone` 和 `limit_req`。

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    limit_req_status 429;

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://127.0.0.1:8080;
    }
}
```

含义：

- 按 IP 限流。
- 平均每秒 10 请求。
- 允许突发 20。
- 超出返回 429。

---

### 57. Nginx 限流和 Go 业务限流有什么区别？

参考回答：

Nginx 适合粗粒度限流：

- 按 IP。
- 按路径。
- 保护入口流量。

Go 适合业务级限流：

- 按用户 ID。
- 按租户。
- 按 API Key。
- 按手机号、验证码场景。

登录、短信、支付等接口通常需要 Nginx + Go 双层保护。

---

### 58. CORS 是什么？Nginx 如何配置？

参考回答：

CORS 是浏览器的跨域访问控制。当前端域名和 API 域名不同，服务端需要返回允许跨域的响应头。

配置：

```nginx
location /api/ {
    add_header Access-Control-Allow-Origin "https://app.example.com" always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;

    if ($request_method = OPTIONS) {
        return 204;
    }

    proxy_pass http://127.0.0.1:8080;
}
```

生产中不建议随便使用 `*`，尤其是有认证的接口。

---

### 59. 什么是 OPTIONS 预检请求？

参考回答：

浏览器发送复杂跨域请求前，会先发送 OPTIONS 请求，询问服务端是否允许实际请求。

例如前端要发送带 Authorization 的 POST 请求，浏览器可能先发：

```http
OPTIONS /api/users
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization
```

Nginx 返回 204 和允许的 CORS 头后，浏览器才会发送真正请求。

---

### 60. Nginx 如何配置代理缓存？

参考回答：

先在 `http` 中定义缓存路径：

```nginx
proxy_cache_path /var/cache/nginx/api_cache
                 levels=1:2
                 keys_zone=api_cache:10m
                 max_size=1g
                 inactive=10m;
```

location 使用：

```nginx
location /api/public/ {
    proxy_cache api_cache;
    proxy_cache_valid 200 1m;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    add_header X-Cache-Status $upstream_cache_status always;
    proxy_pass http://127.0.0.1:8080;
}
```

只适合公共只读数据，不适合用户个人数据、订单、权限相关数据。

---

### 61. 为什么带 Authorization 的请求不应该随便缓存？

参考回答：

带 Authorization 的请求通常和用户身份相关。如果缓存 key 没设计好，可能把 A 用户数据返回给 B 用户。

可以配置：

```nginx
proxy_no_cache $http_authorization;
proxy_cache_bypass $http_authorization;
```

更稳妥的做法是：用户私有数据不要放 Nginx 代理缓存。

---

## 九、性能优化类

### 62. `worker_processes` 和 `worker_connections` 是什么？

参考回答：

`worker_processes` 控制 worker 进程数量：

```nginx
worker_processes auto;
```

`worker_connections` 控制每个 worker 最大连接数：

```nginx
events {
    worker_connections 10240;
}
```

理论最大连接数约等于：

```text
worker_processes * worker_connections
```

但实际还受文件描述符、系统参数、后端连接等限制影响。

---

### 63. 文件描述符限制为什么会影响 Nginx？

参考回答：

Linux 中连接、socket、文件都会占用文件描述符。如果文件描述符上限太低，高并发时可能无法建立更多连接。

查看：

```bash
ulimit -n
cat /proc/$(pgrep -o nginx)/limits | grep "open files"
```

调优时要结合 systemd 的 `LimitNOFILE`、worker 连接数和实际压测结果。

---

### 64. Nginx keepalive 有哪些？

参考回答：

客户端到 Nginx：

```nginx
keepalive_timeout 65;
```

Nginx 到 upstream：

```nginx
upstream go_api {
    server 127.0.0.1:8080;
    keepalive 32;
}

location /api/ {
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_pass http://go_api;
}
```

连接复用能减少频繁建连开销，但不是越大越好，要结合资源限制。

---

### 65. 如何判断性能瓶颈在 Nginx 还是 Go？

参考回答：

先分别压测：

```bash
hey -n 10000 -c 100 http://127.0.0.1:8080/api/ping
hey -n 10000 -c 100 http://api.local/api/ping
```

如果 Go 直连慢，Nginx 代理也慢，瓶颈大概率在 Go 或 Go 的依赖。

如果 Go 直连快，经过 Nginx 慢，需要看 Nginx 配置、TLS、gzip、日志、连接复用等。

再结合 access log：

```text
rt=$request_time
urt=$upstream_response_time
```

如果 `rt` 和 `urt` 都大，慢在 upstream。

---

### 66. gzip 会带来什么性能影响？

参考回答：

gzip 可以减少传输体积，提高弱网加载速度，但压缩会消耗 CPU。

适合压缩：

- HTML。
- CSS。
- JS。
- JSON。

不适合：

- 图片。
- 视频。
- 已压缩文件。

压缩级别不要盲目调太高，要结合 CPU 和响应体大小。

---

## 十、部署与生产实践类

### 67. Nginx 配置如何做工程化管理？

参考回答：

建议：

- 配置拆分。
- 使用数字前缀命名。
- 纳入 Git。
- 发布前 `nginx -t`。
- 发布后 reload。
- 保留回滚版本。
- 有健康检查和验证命令。

示例：

```text
00-log-format.conf
10-upstream.conf
20-app.conf
30-admin.conf
```

不要在生产服务器上随手改配置却没有记录。

---

### 68. systemd 部署 Go + Nginx 的基本方式？

参考回答：

Go 服务由 systemd 管理：

```ini
[Service]
ExecStart=/opt/myapp/myapp
Environment=PORT=8080
Restart=always
```

Nginx 代理：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

发布流程：

```text
构建 Go 二进制
-> 更新文件
-> restart Go 服务
-> nginx -t
-> reload Nginx
-> curl 验证
-> 看日志
```

---

### 69. Docker Compose 中 Nginx 如何访问 Go 容器？

参考回答：

在 Compose 网络中，Nginx 应该通过服务名访问 Go 容器，而不是 `127.0.0.1`。

示例：

```nginx
upstream go_api {
    server app1:8080;
    server app2:8080;
}
```

因为在 Nginx 容器内，`127.0.0.1` 指的是 Nginx 容器自己，不是 Go 容器。

---

### 70. Nginx 配置发布前后应该做哪些检查？

参考回答：

发布前：

```bash
sudo nginx -t
sudo nginx -T
```

发布后：

```bash
sudo systemctl reload nginx
curl -I https://your-domain
sudo tail -n 50 /var/log/nginx/error.log
```

业务验证：

- 首页。
- API。
- HTTPS。
- 静态资源。
- 登录。
- 上传。
- 关键接口。

---

### 71. 如何做 Nginx 配置回滚？

参考回答：

推荐配置纳入 Git。出现问题时切回上一个版本：

```bash
git checkout 上一个稳定版本
sudo nginx -t
sudo systemctl reload nginx
```

如果没有 Git，也至少要发布前备份：

```bash
sudo tar czf nginx-backup-$(date +%Y%m%d-%H%M%S).tar.gz /etc/nginx
```

回滚后必须重新验证业务和日志。

---

### 72. Nginx 和 API Gateway 有什么区别？

参考回答：

Nginx 可以承担一部分网关能力，比如路由、反向代理、HTTPS、限流、日志。

完整 API Gateway 通常还包括：

- 动态路由。
- 服务发现。
- 插件体系。
- 认证鉴权。
- 熔断降级。
- 灰度发布。
- 更完善的监控。

例如 Kong、APISIX、Envoy、Traefik。

小型项目用 Nginx 很合适；复杂微服务系统可能需要 API Gateway 或 Kubernetes Ingress。

---

### 73. Kubernetes Ingress 和 Nginx 有什么关系？

参考回答：

Ingress 是 Kubernetes 中定义外部访问规则的资源。Nginx Ingress Controller 会读取 Ingress 资源，并生成或更新 Nginx 配置。

可以理解为：

```text
Ingress YAML -> Ingress Controller -> Nginx 配置 -> 转发到 Service
```

普通 Nginx 是直接写配置文件；Ingress 是通过 Kubernetes 资源声明路由规则。

---

## 十一、综合场景题

### 74. 用户反馈接口突然变慢，你怎么排查？

参考回答：

我会按链路排查：

1. 先确认是所有接口慢，还是某个接口慢。
2. 看 Nginx access log，关注 `request_time`、`upstream_response_time`。
3. 如果 `upstream_response_time` 高，说明慢在 Go 或后端依赖。
4. 看 Go 日志和 request_id。
5. 查数据库慢查询、Redis、外部 API。
6. 看系统资源 CPU、内存、连接数。
7. 如果 Nginx 代理后才慢，再看 TLS、gzip、keepalive、worker、文件描述符。

不要第一步就调大超时。

---

### 75. 用户反馈上传文件失败，你怎么排查？

参考回答：

先看状态码：

- 413：请求体太大，查 `client_max_body_size`。
- 502：Go 服务不可用或连接失败。
- 504：上传后处理太慢或后端超时。
- 401/403：权限问题。

然后看：

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

检查上传接口是否单独配置：

```nginx
location /api/upload {
    client_max_body_size 100m;
}
```

Go 服务也要检查文件大小限制和处理逻辑。

---

### 76. 线上出现大量 502，你怎么处理？

参考回答：

我会先判断是否后端不可用：

```bash
sudo tail -n 100 /var/log/nginx/error.log
curl http://127.0.0.1:8080/health
ss -lntp | grep 8080
```

如果 error log 是 `Connection refused`，说明 Nginx 连不上后端，检查 Go 服务是否挂了、端口是否变了、容器网络是否异常。

如果只有某个 upstream 出问题，可以临时摘掉问题实例，恢复服务，再看 Go 日志定位根因。

---

### 77. 线上出现大量 504，你怎么处理？

参考回答：

先看日志：

```text
upstream timed out while reading response header from upstream
```

再看 access log：

```text
rt=30.001 urt=30.000
```

说明 Go 响应超过 Nginx 超时。

接着用 request_id 查 Go 日志，看是数据库慢、外部 API 慢、锁等待还是代码阻塞。

只有确认接口确实需要长时间处理，才考虑调整 `proxy_read_timeout`。否则优先优化业务或改成异步任务。

---

### 78. 你会如何设计一个 Go API 的 Nginx 入口配置？

参考回答：

我会考虑：

- HTTP 跳 HTTPS。
- HTTPS server。
- upstream 多实例。
- 基础代理头。
- request_id。
- access log 记录 upstream 耗时。
- `/health` 健康检查。
- `/api/upload` 单独 body size。
- `/api/` 限流。
- 静态资源缓存。
- error log 单独文件。

示例结构：

```text
00-log-format.conf
10-upstream.conf
20-api.conf
```

这样可维护性比较好。

---

### 79. 哪些逻辑适合放 Nginx，哪些适合放 Go？

参考回答：

适合放 Nginx：

- HTTPS。
- 静态资源。
- 反向代理。
- 粗粒度限流。
- 上传大小限制。
- Basic Auth。
- IP 白名单。
- 通用日志。

适合放 Go：

- 用户鉴权。
- 业务权限。
- 订单、支付、库存。
- 幂等。
- 事务。
- 用户级限流。
- 复杂缓存策略。

一句话：

```text
Nginx 负责入口层通用能力，Go 负责业务规则和数据一致性。
```

---

### 80. 你在项目中怎么使用 Nginx？

参考回答：

可以结合自己的项目这样说：

```text
我用 Nginx 作为 Go API 的入口层。Nginx 对外监听 80/443，负责 HTTPS、HTTP 跳转 HTTPS、静态资源服务和 /api/ 反向代理。Go 服务只监听本机或容器内端口。Nginx 通过 upstream 转发到多个 Go 实例，并设置 Host、X-Real-IP、X-Forwarded-For、X-Forwarded-Proto、X-Request-ID 等请求头。日志中记录 request_time、upstream_response_time 和 upstream_addr，用于排查 502、504 和慢请求。另外对登录或 API 路径配置了基础限流，对上传接口单独配置 client_max_body_size。
```

这个回答能体现：

- 你知道 Nginx 在架构中的位置。
- 你知道 Go 服务和 Nginx 的职责边界。
- 你有日志和排障意识。
- 你不是只会复制配置。

---

## 十二、快速复习清单

面试前至少确认你能回答：

1. Nginx 是什么，常见用途是什么？
2. 正向代理和反向代理区别是什么？
3. `server` 和 `location` 如何匹配？
4. `root` 和 `alias` 有什么区别？
5. `try_files` 有什么作用？
6. `proxy_pass` 带 `/` 和不带 `/` 的区别？
7. 常见代理头有哪些？
8. Go 如何获取真实客户端 IP？
9. 为什么不能无条件信任 `X-Forwarded-For`？
10. upstream 如何配置？
11. 轮询、权重、最少连接、ip_hash 的区别？
12. 502 怎么排查？
13. 504 怎么排查？
14. 413 是什么？
15. 499 是什么？
16. HTTPS 如何配置？
17. TLS 终止是什么意思？
18. HTTP 如何跳 HTTPS？
19. `X-Forwarded-Proto` 有什么用？
20. Nginx 如何限流？
21. CORS 如何配置？
22. Nginx 缓存有什么风险？
23. worker 和 worker_connections 是什么？
24. keepalive 有什么作用？
25. 如何判断瓶颈在 Nginx 还是 Go？
26. Nginx 配置如何发布和回滚？
27. Docker Compose 中 Nginx 如何访问 Go 容器？
28. Nginx 和 API Gateway 有什么区别？
29. 哪些逻辑放 Nginx，哪些逻辑放 Go？
30. 如何结合自己的项目讲 Nginx？

