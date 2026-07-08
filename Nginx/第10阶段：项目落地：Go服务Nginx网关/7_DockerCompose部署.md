# 7. Docker Compose 部署

本节目标：使用 Docker Compose 启动 Go 服务和 Nginx，理解容器网络中 Nginx 如何通过服务名访问 Go。

---

## 一、Compose 项目结构

```text
nginx-gateway-demo/
  docker-compose.yml
  app/
    Dockerfile
    go.mod
    main.go
  static/
    index.html
    assets/
      style.css
  nginx/
    nginx.conf
    conf.d/
      00-log-format.conf
      10-upstream.conf
      20-app.conf
```

---

## 二、Go Dockerfile

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

---

## 三、Compose 文件

`docker-compose.yml`：

```yaml
services:
  app1:
    build:
      context: ./app
    environment:
      PORT: "8080"
    expose:
      - "8080"

  app2:
    build:
      context: ./app
    environment:
      PORT: "8080"
    expose:
      - "8080"

  app3:
    build:
      context: ./app
    environment:
      PORT: "8080"
    expose:
      - "8080"

  nginx:
    image: nginx:stable
    ports:
      - "8088:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./static:/var/www/app/static:ro
    depends_on:
      - app1
      - app2
      - app3
```

这里把宿主机 `8088` 映射到容器内 Nginx `80`，避免占用本机 80。

---

## 四、容器版 upstream

容器里不能写：

```nginx
server 127.0.0.1:8080;
```

因为在 Nginx 容器中，`127.0.0.1` 指的是 Nginx 容器自己。

应该写 Compose 服务名：

`nginx/conf.d/10-upstream.conf`：

```nginx
upstream go_api {
    server app1:8080;
    server app2:8080;
    server app3:8080;
    keepalive 32;
}
```

---

## 五、启动

```bash
docker compose up -d --build
```

查看：

```bash
docker compose ps
docker compose logs -f nginx
docker compose logs -f app1
```

验证：

```bash
curl -H "Host: app.local" http://127.0.0.1:8088/api/ping
```

---

## 六、常见问题

### 1. Nginx 连接 app 失败

检查：

```bash
docker compose ps
docker compose logs app1
docker compose exec nginx nginx -T
```

确认 upstream 使用的是：

```text
app1:8080
app2:8080
app3:8080
```

而不是 `127.0.0.1:8080`。

### 2. 本机 8088 访问失败

检查：

```bash
docker compose ps
```

确认端口映射：

```text
0.0.0.0:8088->80/tcp
```

---

## 七、本节练习

1. 创建 Go Dockerfile。
2. 创建 docker-compose.yml。
3. 修改 upstream 使用 app1、app2、app3。
4. 启动 Compose。
5. 连续请求 `/api/ping` 验证负载均衡。
6. 查看 Nginx 和 Go 容器日志。

---

## 八、本节复盘

请确认你能回答：

1. 容器里的 `127.0.0.1` 指向哪里？
2. Nginx 容器为什么要用 `app1:8080` 访问 Go？
3. `ports` 和 `expose` 有什么区别？
4. 配置文件为什么用只读挂载？
5. 如何查看 Nginx 容器中的最终配置？

