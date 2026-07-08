# Go 后端工程师的 Nginx 系统学习路线图

> 目标：不是只会背 Nginx 配置，而是能把 Nginx 放进真实后端系统里，完成反向代理、负载均衡、HTTPS、静态资源、网关限流、日志排障、性能调优和线上部署。

## 一、学习前置基础

在正式学习 Nginx 前，建议先补齐这些基础。它们会直接影响你理解 Nginx 的配置、网络行为和线上问题。

### 1. Linux 基础

你需要掌握：

- 常用命令：`cd`、`ls`、`cat`、`tail`、`grep`、`find`、`ps`、`netstat`、`ss`、`curl`
- 文件权限：`chmod`、`chown`、用户、用户组
- 进程管理：查看进程、杀进程、后台运行、服务管理
- systemd：`systemctl start|stop|restart|status nginx`
- 日志查看：`tail -f`、`journalctl`

练习目标：

- 能在 Linux 上安装并启动 Nginx
- 能找到 Nginx 的配置文件、日志文件、运行用户和进程

### 2. 计算机网络基础

重点理解：

- IP、端口、DNS
- TCP 三次握手和四次挥手
- HTTP 请求与响应
- HTTP Header、状态码、Cookie
- HTTPS、TLS、证书
- 正向代理与反向代理

练习目标：

- 能用 `curl -v` 分析一次 HTTP 请求
- 能解释浏览器访问一个域名后发生了什么
- 能区分 `502`、`503`、`504` 的常见原因

### 3. Go Web 服务基础

你需要能写一个简单 Go HTTP 服务：

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "pong")
	})

	http.ListenAndServe(":8080", nil)
}
```

练习目标：

- 启动多个 Go 服务实例，例如 `:8080`、`:8081`、`:8082`
- 用 Nginx 把请求代理到 Go 服务
- 理解 Nginx 在 Go 后端前面承担什么角色

## 二、第一阶段：Nginx 入门与基础使用

### 学习目标

你要先做到：能安装、启动、修改配置、检查配置、重载配置，并看懂最基本的 Nginx 配置结构。

### 核心知识

- Nginx 是什么
- Nginx 常见使用场景
- Nginx 进程模型：master 与 worker
- 配置文件结构：`main`、`events`、`http`、`server`、`location`
- 常用命令：
  - `nginx -t`
  - `nginx -s reload`
  - `systemctl restart nginx`
  - `systemctl status nginx`

### 推荐练习

1. 安装 Nginx。
2. 启动 Nginx 并访问默认页面。
3. 修改默认监听端口。
4. 配置一个最简单的静态页面。
5. 故意写错配置，使用 `nginx -t` 查看错误。

### 阶段产出

你应该能写出类似下面的配置：

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

## 三、第二阶段：静态资源服务

### 学习目标

掌握 Nginx 托管静态文件的能力，理解 `root`、`alias`、`index`、`try_files` 的区别。

### 核心知识

- `root` 与 `alias`
- `index`
- `try_files`
- 目录访问控制
- 静态资源缓存
- gzip 压缩
- MIME 类型

### 推荐练习

1. 使用 Nginx 托管一个 HTML 页面。
2. 托管 CSS、JS、图片等静态资源。
3. 分别使用 `root` 和 `alias` 实现文件访问。
4. 配置静态资源缓存。
5. 开启 gzip 压缩。

### 常见配置示例

```nginx
server {
    listen 80;
    server_name static.local;

    location / {
        root /var/www/site;
        index index.html;
        try_files $uri $uri/ =404;
    }

    location /assets/ {
        alias /var/www/site/assets/;
        expires 7d;
        add_header Cache-Control "public";
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
}
```

### 阶段产出

你应该能独立部署一个前端静态页面，并知道访问不到文件时如何排查路径问题。

## 四、第三阶段：反向代理 Go 后端服务

### 学习目标

这是 Go 后端工程师学习 Nginx 的核心阶段。你要理解 Nginx 如何把客户端请求转发给 Go 服务。

### 核心知识

- 反向代理的作用
- `proxy_pass`
- `proxy_set_header`
- 请求头透传
- 客户端真实 IP
- 超时配置
- 请求体大小限制
- WebSocket 代理

### 推荐练习

1. 启动一个 Go HTTP 服务，监听 `8080`。
2. 配置 Nginx 监听 `80`，把请求转发到 Go 服务。
3. 在 Go 服务中打印请求头，观察 Nginx 转发后的变化。
4. 配置 `X-Real-IP`、`X-Forwarded-For`。
5. 修改 Go 服务路径，验证 `proxy_pass` 后面带不带 `/` 的区别。

### 常见配置示例

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

        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

### Go 后端重点关注

在 Go 服务里，很多线上逻辑会依赖代理头：

- 获取真实客户端 IP
- 判断原始请求协议是 HTTP 还是 HTTPS
- 生成回调 URL
- 做限流、审计、安全日志

你需要清楚：Go 服务看到的直接客户端通常是 Nginx，而不是用户浏览器。

### 阶段产出

你应该能使用 Nginx 代理 Go API 服务，并能解释每个代理头的作用。

## 五、第四阶段：负载均衡与高可用基础

### 学习目标

掌握 Nginx 如何把流量分发到多个 Go 服务实例。

### 核心知识

- `upstream`
- 轮询
- 权重
- 最少连接
- IP Hash
- 后端失败重试
- 健康检查的基础思路

### 推荐练习

1. 启动三个 Go 服务实例：`8080`、`8081`、`8082`。
2. 每个实例返回不同的实例编号。
3. 使用 Nginx `upstream` 做负载均衡。
4. 停掉一个实例，观察请求变化。
5. 配置权重，观察流量比例变化。

### 常见配置示例

```nginx
upstream go_api {
    server 127.0.0.1:8080 weight=3;
    server 127.0.0.1:8081 weight=1;
    server 127.0.0.1:8082 weight=1;
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

### 阶段产出

你应该能回答：

- Nginx 如何把请求分发给多个 Go 实例？
- 某个 Go 实例挂掉时会发生什么？
- 为什么实际生产中还需要更完整的健康检查和服务发现？

## 六、第五阶段：HTTPS 与安全配置

### 学习目标

掌握 HTTPS 证书配置，并理解 Nginx 在 TLS 终止中的作用。

### 核心知识

- HTTP 与 HTTPS 的区别
- TLS 握手基础
- 证书、公钥、私钥、CA
- Let's Encrypt
- HTTP 自动跳转 HTTPS
- TLS 终止
- 安全响应头
- 隐藏 Nginx 版本号

### 推荐练习

1. 使用自签名证书配置 HTTPS。
2. 使用 Let's Encrypt 申请真实证书。
3. 配置 HTTP 跳转 HTTPS。
4. 给 Go 服务只暴露内网端口，由 Nginx 对外提供 HTTPS。
5. 配置基础安全响应头。

### 常见配置示例

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-XSS-Protection "1; mode=block";

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### 阶段产出

你应该能把一个 Go API 服务通过 HTTPS 安全地暴露到公网。

## 七、第六阶段：日志、监控与排障

### 学习目标

能通过 Nginx 日志定位常见线上问题。

### 核心知识

- `access.log`
- `error.log`
- 自定义日志格式
- 请求耗时
- 上游服务耗时
- 状态码分析
- 502、503、504 排查
- `curl`、`ss`、`lsof`、`journalctl` 辅助排查

### 推荐练习

1. 自定义 Nginx 日志格式，记录上游响应时间。
2. 故意让 Go 服务返回慢响应，观察日志。
3. 停掉 Go 服务，观察 Nginx 返回的状态码。
4. 修改错误端口，观察 `error.log`。
5. 使用 `tail -f` 实时分析请求。

### 常见日志配置

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" '
                'rt=$request_time '
                'uct=$upstream_connect_time '
                'uht=$upstream_header_time '
                'urt=$upstream_response_time';

access_log /var/log/nginx/access.log main;
error_log /var/log/nginx/error.log warn;
```

### 排障思路

遇到问题时，按这个顺序查：

1. 客户端请求是否正确：域名、路径、Header、Body。
2. Nginx 是否收到请求：看 `access.log`。
3. Nginx 配置是否正确：`nginx -t`。
4. Nginx 是否能连上 Go 服务：端口、进程、防火墙。
5. Go 服务是否正常响应：直接 `curl 127.0.0.1:8080`。
6. 响应是否超时：看 Nginx 超时配置和 Go 服务日志。

### 阶段产出

你应该能根据 Nginx 日志判断问题大概率出在客户端、Nginx、网络还是 Go 服务。

## 八、第七阶段：限流、缓存与网关能力

### 学习目标

理解 Nginx 不只是转发工具，还可以承担部分网关能力。

### 核心知识

- 请求限流：`limit_req`
- 连接限制：`limit_conn`
- IP 黑白名单
- Basic Auth
- 代理缓存：`proxy_cache`
- 大文件上传限制
- 跨域配置 CORS

### 推荐练习

1. 对 `/api/` 做基础限流。
2. 限制单个 IP 的并发连接数。
3. 给管理后台加 Basic Auth。
4. 配置上传大小限制。
5. 对某个只读接口做短时间缓存。

### 常见限流配置

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        listen 80;
        server_name api.local;

        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;

            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

### Go 后端重点关注

Nginx 限流适合做粗粒度保护，例如：

- 防止单 IP 突发请求打爆服务
- 保护登录、短信、验证码接口
- 降低恶意爬虫压力

复杂业务限流仍然建议在 Go 服务或专门网关中实现，例如按用户 ID、租户 ID、API Key 维度限流。

### 阶段产出

你应该能为 Go API 增加一层基础防护。

## 九、第八阶段：性能调优

### 学习目标

理解 Nginx 常见性能参数，知道什么时候该调、怎么验证。

### 核心知识

- `worker_processes`
- `worker_connections`
- `keepalive_timeout`
- `client_body_buffer_size`
- `client_max_body_size`
- `sendfile`
- `tcp_nopush`
- `gzip`
- 上游连接复用
- 文件描述符限制

### 推荐练习

1. 使用压测工具压测 Go 服务直连性能。
2. 使用压测工具压测 Nginx 代理后的性能。
3. 观察 CPU、内存、连接数和错误日志。
4. 调整 worker 配置并比较差异。
5. 分析 Nginx 是否成为瓶颈。

### 基础优化示例

```nginx
worker_processes auto;

events {
    worker_connections 10240;
}

http {
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    upstream go_api {
        server 127.0.0.1:8080;
        keepalive 32;
    }

    server {
        listen 80;
        server_name api.local;

        location / {
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_pass http://go_api;
        }
    }
}
```

### 阶段产出

你应该能说清楚：Nginx 性能问题需要结合系统资源、连接数、请求耗时和 Go 服务表现一起分析。

## 十、第九阶段：部署与生产实践

### 学习目标

掌握真实生产环境中 Nginx 与 Go 服务的部署方式。

### 核心知识

- 单机部署
- 多实例部署
- Docker 部署
- Docker Compose
- Nginx 与 Go 服务容器网络
- 配置文件拆分
- 灰度发布基础
- 零停机重载
- 配置备份与回滚

### 推荐练习

1. 用 systemd 管理 Go 服务。
2. 用 Nginx 代理 systemd 启动的 Go 服务。
3. 使用 Docker Compose 启动 Nginx 和 Go 服务。
4. 把 Nginx 配置拆成 `conf.d/*.conf`。
5. 模拟发布新版本并回滚。

### Docker Compose 示例方向

你可以设计这样的结构：

```text
project/
  docker-compose.yml
  nginx/
    nginx.conf
    conf.d/
      api.conf
  app/
    Dockerfile
    main.go
```

### 阶段产出

你应该能独立部署一个基于 Go + Nginx 的小型后端服务。

## 十一、第十阶段：深入理解 Nginx

### 学习目标

从会用走向理解原理，能阅读更复杂的配置并进行架构判断。

### 核心知识

- Nginx 事件驱动模型
- 多进程模型
- 非阻塞 I/O
- Nginx 配置继承规则
- location 匹配规则
- rewrite 规则
- 变量机制
- 请求处理阶段
- OpenResty 基础了解
- Nginx 与 API Gateway 的边界

### 推荐练习

1. 系统学习 `location` 匹配优先级。
2. 写多个 `location`，验证匹配结果。
3. 使用 `rewrite` 做路径重写。
4. 阅读一份真实公司的 Nginx 配置。
5. 对比 Nginx、Kong、Traefik、Envoy 的定位。

### 阶段产出

你应该能看懂中大型项目里的 Nginx 配置，并知道哪些逻辑适合放在 Nginx，哪些逻辑应该放在 Go 服务。

## 十二、推荐实战项目

### 项目 1：Go API 反向代理

目标：

- 写一个 Go API 服务。
- 使用 Nginx 代理。
- 配置真实 IP。
- 配置日志。
- 配置超时。

验收标准：

- 浏览器或 `curl` 访问 Nginx，能拿到 Go 服务响应。
- Go 服务能打印真实客户端信息。
- Nginx 日志中能看到请求耗时和上游耗时。

### 项目 2：多实例负载均衡

目标：

- 启动 3 个 Go 服务实例。
- Nginx 做负载均衡。
- 支持权重配置。
- 模拟实例故障。

验收标准：

- 多次请求能分发到不同实例。
- 停掉一个实例后，请求仍能访问成功。
- 能解释错误日志中的连接失败信息。

### 项目 3：HTTPS Go 服务部署

目标：

- 使用域名访问服务。
- 配置 HTTPS。
- HTTP 自动跳转 HTTPS。
- Go 服务只监听本地或内网端口。

验收标准：

- 外部只能通过 HTTPS 访问。
- 证书有效。
- Go 服务无需直接暴露公网端口。

### 项目 4：Nginx 网关基础能力

目标：

- 对接口做限流。
- 添加 CORS。
- 配置上传大小限制。
- 配置管理后台 Basic Auth。

验收标准：

- 高频请求会被限制。
- 跨域请求符合预期。
- 超大请求体会被拒绝。
- 管理路径需要认证。

## 十三、建议学习节奏

### 第 1 周：基础入门

- 安装 Nginx
- 学习配置结构
- 部署静态页面
- 熟悉常用命令

### 第 2 周：反向代理 Go 服务

- 写 Go HTTP 服务
- 配置 `proxy_pass`
- 学习代理头
- 学习超时配置

### 第 3 周：负载均衡与 HTTPS

- 配置 `upstream`
- 启动多个 Go 实例
- 配置 HTTPS
- 学习证书与 TLS 终止

### 第 4 周：日志与排障

- 自定义日志格式
- 分析 502、503、504
- 使用 `curl` 和系统命令排查
- 故意制造错误并修复

### 第 5 周：限流、缓存、安全

- 配置限流
- 配置 IP 访问控制
- 配置缓存
- 配置安全响应头

### 第 6 周：部署与综合项目

- systemd 部署 Go 服务
- Docker Compose 部署 Go + Nginx
- 完成一个完整后端服务部署项目
- 整理自己的 Nginx 配置模板

## 十四、Go 后端工程师需要重点掌握的 Nginx 能力

优先级从高到低：

1. 反向代理 Go 服务。
2. 正确传递客户端真实 IP 和协议。
3. 配置负载均衡。
4. 配置 HTTPS。
5. 看懂 access log 和 error log。
6. 排查 502、503、504。
7. 配置超时和请求体大小。
8. 配置基础限流和安全头。
9. 部署静态资源和前端页面。
10. 理解 Nginx 与 Go 服务的职责边界。

## 十五、常见坑位清单

### 1. `proxy_pass` 路径问题

`proxy_pass http://127.0.0.1:8080;` 和 `proxy_pass http://127.0.0.1:8080/;` 在路径转发上有差异，尤其和 `location /api/` 搭配时要特别注意。

### 2. Go 服务拿不到真实 IP

如果没有配置：

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Go 服务通常只能看到 Nginx 的 IP。

### 3. 线上出现 502

常见原因：

- Go 服务没启动
- 端口写错
- Go 服务崩溃
- Nginx 没权限访问 upstream
- Docker 网络配置错误

### 4. 线上出现 504

常见原因：

- Go 服务响应太慢
- 数据库慢查询
- Nginx `proxy_read_timeout` 太短
- 后端 goroutine 阻塞

### 5. 配置改了没生效

常见原因：

- 忘记 `nginx -t`
- 忘记 reload
- 修改了错误的配置文件
- 配置文件没有被 `include`

## 十六、推荐资料

### 官方资料

- Nginx 官方文档：https://nginx.org/en/docs/
- Nginx 变量索引：https://nginx.org/en/docs/varindex.html
- Nginx 指令索引：https://nginx.org/en/docs/dirindex.html

### 实战方向资料

- Linux 网络命令学习
- HTTP 权威指南相关章节
- Go `net/http` 官方文档
- Docker Compose 官方文档
- Let's Encrypt 与 Certbot 文档

## 十七、最终能力检验

当你完成这条路线后，尝试独立完成下面这个任务：

> 在一台 Linux 服务器上部署一个 Go REST API 服务，使用 Nginx 对外提供 HTTPS 访问，支持静态前端页面、API 反向代理、三个 Go 实例负载均衡、请求日志、错误日志、基础限流、上传大小限制，并能排查 502 和 504 问题。

如果你能完成这个任务，并能解释每一段 Nginx 配置的作用，说明你已经具备 Go 后端工程师日常工作中所需的 Nginx 能力。

## 十八、建议沉淀自己的模板

学习过程中，建议你逐步整理这些模板：

- `static-site.conf`：静态站点模板
- `go-api-proxy.conf`：Go API 反向代理模板
- `go-upstream.conf`：Go 多实例负载均衡模板
- `https-site.conf`：HTTPS 模板
- `logging.conf`：日志格式模板
- `rate-limit.conf`：限流模板
- `docker-compose-nginx-go.yml`：Docker Compose 部署模板

这些模板会成为你以后部署项目时的工程资产。
