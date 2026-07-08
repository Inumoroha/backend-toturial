# 09-01 Docker Compose、Nginx 与部署检查

本阶段目标：把 Gin 服务从“本地能运行”推进到“可以部署”。你会学习编译 Go 程序、编写 Dockerfile、使用 Docker Compose 启动依赖，并了解 Nginx 反向代理和生产检查。

## 一、编译 Go 程序

Go 可以编译成单个可执行文件：

```bash
go build -o bin/server ./cmd/server
```

运行：

```bash
./bin/server
```

Windows 下可能是：

```bash
.\bin\server.exe
```

交叉编译 Linux 程序：

```bash
GOOS=linux GOARCH=amd64 go build -o bin/server ./cmd/server
```

Windows PowerShell：

```powershell
$env:GOOS="linux"
$env:GOARCH="amd64"
go build -o bin/server ./cmd/server
```

## 二、健康检查接口

部署前先加健康检查：

```go
r.GET("/healthz", func(c *gin.Context) {
    c.JSON(200, gin.H{
        "status": "ok",
    })
})
```

如果要检查数据库：

```go
sqlDB, _ := db.DB()
if err := sqlDB.Ping(); err != nil {
    c.JSON(503, gin.H{"status": "database unavailable"})
    return
}
```

健康检查可以给 Docker、负载均衡和监控系统使用。

## 三、Dockerfile

推荐使用多阶段构建。

创建：

```text
Dockerfile
```

内容：

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o server ./cmd/server

FROM alpine:3.20

WORKDIR /app

COPY --from=builder /app/server /app/server
COPY configs /app/configs

EXPOSE 8080

CMD ["/app/server"]
```

构建镜像：

```bash
docker build -t gin-learning:latest .
```

运行：

```bash
docker run --rm -p 8080:8080 gin-learning:latest
```

## 四、Docker Compose

创建：

```text
docker-compose.yml
```

示例：

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
      - redis
    environment:
      APP_ENV: "local"

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: gin_learning
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  mysql_data:
```

启动：

```bash
docker compose up -d
```

查看日志：

```bash
docker compose logs -f app
```

停止：

```bash
docker compose down
```

如果要删除数据库数据卷：

```bash
docker compose down -v
```

注意这个命令会删除数据，开发环境可以用，生产环境不要随便执行。

## 五、容器内连接 MySQL

在 Docker Compose 网络中，应用连接 MySQL 不应该使用 `127.0.0.1`，而应该使用服务名：

```text
root:password@tcp(mysql:3306)/gin_learning?charset=utf8mb4&parseTime=True&loc=Local
```

连接 Redis：

```text
redis:6379
```

这是很多初学者会踩的坑。

## 六、Nginx 反向代理

Nginx 配置示例：

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

作用：

- 对外提供 80/443 端口。
- 把请求转发给 Gin 服务。
- 统一处理 HTTPS、静态文件、访问日志等。

生产环境建议使用 HTTPS。

## 七、生产环境检查清单

上线前至少检查：

- 配置是否从环境变量或配置文件读取。
- 数据库密码、JWT secret 是否没有写死在代码里。
- Gin 是否使用 release 模式。
- 日志是否可查看。
- 健康检查接口是否可访问。
- 数据库迁移是否完成。
- Docker 镜像是否能独立启动。
- Redis、MySQL 地址是否适合容器网络。
- 上传文件是否有持久化策略。
- 是否有备份方案。

Gin release 模式：

```bash
GIN_MODE=release
```

或代码中：

```go
gin.SetMode(gin.ReleaseMode)
```

更推荐用环境变量控制。

## 八、常见问题

### 1. 容器启动后连不上数据库

检查 DSN 是否用了 `mysql:3306`，而不是 `127.0.0.1:3306`。

### 2. app 比 mysql 先启动

`depends_on` 只保证启动顺序，不保证 MySQL 已经可用。应用可以增加连接重试，或者在启动脚本里等待数据库。

### 3. 容器内没有配置文件

确认 Dockerfile 中有：

```dockerfile
COPY configs /app/configs
```

或者通过 volume 挂载配置。

## 九、阶段练习

完成：

- 为项目增加 `/healthz`。
- 编写 Dockerfile。
- 编写 docker-compose.yml。
- 使用 Compose 启动 Gin、MySQL、Redis。
- 确认接口能访问。
- 写一份部署说明到项目 README。

## 十、验收清单

你应该能够回答：

- Go 程序为什么适合容器化？
- Dockerfile 多阶段构建有什么好处？
- Compose 中 app 连接 MySQL 为什么用服务名？
- 健康检查接口有什么用？
- Nginx 反向代理解决什么问题？

