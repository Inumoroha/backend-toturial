# 02 生产部署：Docker Compose 部署 Nginx + Go

## 本节目标

学习用 Docker Compose 同时管理 Go 服务和 Nginx。这个方式适合学习容器网络、快速搭建环境，也适合小型项目部署。

## 一、推荐项目结构

```text
project/
  docker-compose.yml
  app/
    Dockerfile
    go.mod
    main.go
  nginx/
    nginx.conf
    conf.d/
      api.conf
```

## 二、Go 服务 Dockerfile

`app/Dockerfile`：

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/app .

FROM alpine:3.20
WORKDIR /app
COPY --from=builder /out/app /app/app
EXPOSE 8080
CMD ["/app/app"]
```

如果你的项目没有外部依赖，`go mod download` 可能没有太多输出，这是正常的。

## 三、Nginx 配置

`nginx/nginx.conf`：

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    include /etc/nginx/conf.d/*.conf;
}
```

`nginx/conf.d/api.conf`：

```nginx
upstream go_api {
    server app:8080;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://go_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

注意：在 Compose 网络里，`app` 是服务名，可以被 Nginx 当作域名解析。

## 四、docker-compose.yml

```yaml
services:
  app:
    build:
      context: ./app
    expose:
      - "8080"

  nginx:
    image: nginx:stable
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    depends_on:
      - app
```

启动：

```bash
docker compose up -d --build
```

查看：

```bash
docker compose ps
docker compose logs -f nginx
docker compose logs -f app
```

验证：

```bash
curl http://127.0.0.1/api/ping
```

## 五、扩容 Go 服务

```bash
docker compose up -d --scale app=3
```

如果使用 `server app:8080;`，Nginx 对 Docker DNS 的动态变化支持有限，实际生产要更仔细设计服务发现。学习阶段可以先理解容器网络和服务名访问。

## 六、常见问题

### Nginx 连接 app 失败

检查：

```bash
docker compose ps
docker compose logs app
docker compose exec nginx nginx -T
```

确认：

- Go 服务容器是否启动。
- Go 服务是否监听 `:8080`，而不是只监听 `127.0.0.1:8080`。
- Nginx 配置中 upstream 是否写成 `app:8080`。

### 端口冲突

如果本机已有 Nginx 占用 80，可以改成：

```yaml
ports:
  - "8088:80"
```

访问：

```bash
curl http://127.0.0.1:8088/api/ping
```

## 七、本节练习

1. 创建 Compose 项目结构。
2. 构建 Go 服务镜像。
3. 让 Nginx 代理到 `app:8080`。
4. 查看两个容器日志。
5. 修改 Go 代码后重新 build 并发布。

## 八、你应该掌握

学完本节，你应该能解释：

- 容器里为什么不能写 `127.0.0.1:8080` 访问另一个容器。
- Compose 服务名如何用于容器间通信。
- Nginx 配置如何挂载进容器。

