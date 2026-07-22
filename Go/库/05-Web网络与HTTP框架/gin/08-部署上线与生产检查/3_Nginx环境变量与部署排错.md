# 3. Nginx、环境变量与部署排错入口

本节目标：使用 Nginx 反向代理 Gin 服务，并建立部署排错的基本入口。

在真实部署中，Gin 应用通常不会直接暴露给公网用户，而是放在 Nginx、负载均衡或网关后面。Nginx 可以处理域名、HTTPS、静态资源、反向代理和部分请求限制。

---

## 一、Nginx 在这里做什么

本阶段先让 Nginx 完成最基础职责：

```text
客户端
  ↓
Nginx :80
  ↓
Gin app :8080
```

Nginx 接收外部请求，再转发给 app 容器。

这样做的好处：

- 外部只暴露 80 或 443。
- 后端服务端口不用直接暴露公网。
- 后续可以加 HTTPS。
- 可以统一处理静态文件、请求体大小、代理头。

---

## 二、编写 nginx.conf

项目根目录创建 `nginx.conf`：

```nginx
events {}

http {
    upstream gin_app {
        server app:8080;
    }

    server {
        listen 80;
        server_name _;

        client_max_body_size 5m;

        location / {
            proxy_pass http://gin_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

关键点：

- `server app:8080`：Compose 网络中使用 app 服务名。
- `client_max_body_size 5m`：限制请求体大小，和上传限制配合。
- `proxy_set_header`：把真实客户端信息传给后端。

---

## 三、在 Compose 中加入 Nginx

修改 `docker-compose.yml`：

```yaml
  nginx:
    image: nginx:1.27-alpine
    container_name: gin_nginx
    depends_on:
      - app
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

启动：

```bash
docker compose up -d --build
```

访问：

```bash
curl http://localhost/ping
```

如果返回正常，说明 Nginx 已经代理到 Gin。

---

## 四、为什么 Nginx 里写 app:8080

Nginx 和 app 在同一个 Compose 网络中，所以 Nginx 可以通过服务名访问 app。

这里不是写：

```text
localhost:8080
```

因为在 Nginx 容器中，`localhost` 指的是 Nginx 容器自己。

这和 app 连接 MySQL、Redis 时不能写 `127.0.0.1` 是同一个道理。

---

## 五、Gin 获取真实 IP

如果有 Nginx 代理，Gin 看到的直接来源可能是 Nginx 容器 IP。

Nginx 已经设置：

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Gin 中如果使用 `c.ClientIP()`，需要正确配置可信代理。学习阶段可以先理解这个问题，生产环境要根据部署网络设置：

```go
r.SetTrustedProxies([]string{"172.16.0.0/12"})
```

或者在明确场景下设置具体代理 IP。

不要在不了解网络边界时随便信任所有代理。

---

## 六、Nginx 502 怎么排查

502 表示 Nginx 没能成功从上游 app 拿到响应。

排查顺序：

```bash
docker compose ps
docker compose logs nginx
docker compose logs app
```

重点检查：

- app 容器是否运行。
- app 是否监听 `0.0.0.0:8080`，而不是只监听 `127.0.0.1`。
- Nginx upstream 是否写成 `app:8080`。
- app 是否启动失败。

Gin 服务在容器中应该监听：

```text
:8080
```

不要只绑定：

```text
127.0.0.1:8080
```

---

## 七、上传接口和 Nginx 限制

如果 Gin handler 允许最大头像 2MB，但 Nginx 默认请求体限制较小，可能会先被 Nginx 拒绝。

配置：

```nginx
client_max_body_size 5m;
```

建议 Nginx 限制略大于应用限制。这样：

- 特别大的请求可以在 Nginx 层拦截。
- 应用层仍负责业务错误响应。

---

## 八、常见问题

### 1. 访问 localhost 还是到不了 app

检查 Nginx 是否启动，端口是否映射：

```bash
docker compose ps nginx
```

### 2. Nginx 配置改了不生效

重启 Nginx：

```bash
docker compose restart nginx
```

或者重新加载：

```bash
docker compose exec nginx nginx -s reload
```

### 3. app 日志里客户端 IP 都是 Nginx

检查代理头和 Gin trusted proxies 配置。

---

## 九、练习

请完成：

1. 编写 `nginx.conf`。
2. Compose 中加入 nginx 服务。
3. 通过 `http://localhost/ping` 访问 Gin。
4. 故意停止 app，观察 Nginx 502。
5. 查看 Nginx 和 app 日志。

---

## 十、验收标准

完成后确认：

- Nginx 可以启动。
- Nginx 能代理到 app。
- Nginx 中 upstream 使用服务名 `app`。
- app 容器不需要直接暴露给公网也能被 Nginx 访问。
- 你知道 502 应该从哪些日志开始查。

下一节专门补健康检查和服务状态接口。

