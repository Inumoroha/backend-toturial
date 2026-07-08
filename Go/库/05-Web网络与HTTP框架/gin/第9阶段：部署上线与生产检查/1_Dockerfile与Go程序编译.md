# 1. Dockerfile 与 Go 程序编译

本节目标：为 Gin 项目编写 Dockerfile，把 Go 程序构建成可以运行的容器镜像。

Dockerfile 的本质是描述“如何从源码得到一个可运行环境”。Go 项目很适合容器化，因为 Go 可以编译成独立二进制文件。

---

## 一、先确认本地能编译

在写 Dockerfile 前，先确保项目本地可编译：

```bash
go test ./...
go build -o bin/server ./cmd/server
```

运行：

```bash
./bin/server -config=config/config.local.yaml
```

Windows PowerShell：

```powershell
go build -o bin/server.exe ./cmd/server
.\bin\server.exe -config=config/config.local.yaml
```

如果本地都不能编译，先不要急着写 Dockerfile。Docker 只会把问题包起来，不会让问题消失。

---

## 二、为什么使用多阶段构建

不推荐直接用一个 Go 镜像运行：

```dockerfile
FROM golang:1.22
COPY . .
RUN go build ...
CMD ["./server"]
```

这样镜像里会包含 Go 编译器、模块缓存、源码等，体积大，也不够干净。

多阶段构建分成：

```text
builder 阶段：负责编译
runtime 阶段：只放运行需要的二进制和配置
```

---

## 三、编写 Dockerfile

项目根目录创建 `Dockerfile`：

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app

RUN apk add --no-cache git ca-certificates

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /app/server ./cmd/server

FROM alpine:3.20

WORKDIR /app

RUN apk add --no-cache ca-certificates tzdata

COPY --from=builder /app/server /app/server
COPY config /app/config

RUN mkdir -p /app/uploads/avatars

EXPOSE 8080

CMD ["/app/server", "-config=/app/config/config.yaml"]
```

---

## 四、逐行解释

```dockerfile
FROM golang:1.22-alpine AS builder
```

使用 Go Alpine 镜像作为编译环境，并命名为 builder。

```dockerfile
WORKDIR /app
```

设置容器内工作目录。

```dockerfile
COPY go.mod go.sum ./
RUN go mod download
```

先复制依赖文件并下载模块，可以利用 Docker 缓存。只要依赖不变，后续构建会更快。

```dockerfile
COPY . .
```

复制项目源码。

```dockerfile
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build
```

编译 Linux amd64 可执行文件，并关闭 CGO，方便在轻量镜像中运行。

```dockerfile
FROM alpine:3.20
```

运行阶段使用更小的 Alpine 镜像。

```dockerfile
COPY --from=builder /app/server /app/server
```

只从 builder 阶段复制编译好的二进制。

---

## 五、编写 .dockerignore

项目根目录创建 `.dockerignore`：

```dockerignore
.git
.idea
.vscode
bin
tmp
logs
uploads
docs
*.log
.env
```

作用：

- 减少构建上下文。
- 避免把本地临时文件复制进镜像。
- 避免真实 `.env` 进入镜像。

注意：如果你希望镜像中包含 Swagger docs，就不要忽略 `docs`，或者在构建前生成后复制进去。学习阶段可根据项目需要调整。

---

## 六、构建镜像

执行：

```bash
docker build -t gin-user-api:dev .
```

查看镜像：

```bash
docker images gin-user-api
```

运行：

```bash
docker run --rm -p 8080:8080 gin-user-api:dev
```

访问：

```bash
curl http://localhost:8080/ping
```

---

## 七、容器中 localhost 的坑

如果容器里的配置写：

```yaml
database:
  dsn: "root:password@tcp(127.0.0.1:3306)/app"
redis:
  addr: "127.0.0.1:6379"
```

通常会连接失败。

原因：容器里的 `127.0.0.1` 指的是当前 app 容器，不是你的宿主机，也不是 MySQL 容器。

Docker Compose 中应该使用服务名：

```text
mysql:3306
redis:6379
```

这个问题会在下一节详细处理。

---

## 八、常见问题

### 1. `go mod download` 很慢

可以配置 Go 代理：

```dockerfile
ENV GOPROXY=https://goproxy.cn,direct
```

也可以使用默认代理。生产环境要根据团队规范选择。

### 2. 容器运行提示 permission denied

检查二进制是否正确复制，是否有执行权限。Go build 产物一般可执行。

### 3. 容器启动后马上退出

查看日志：

```bash
docker logs 容器ID
```

常见原因是配置文件路径错误、数据库连接失败、端口被占用。

---

## 九、练习

请完成：

1. 编写多阶段 Dockerfile。
2. 编写 `.dockerignore`。
3. 构建 `gin-user-api:dev` 镜像。
4. 使用 `docker run` 启动容器。
5. 访问 `/ping` 验证服务。

---

## 十、验收标准

本节完成后确认：

- 本地 `go build` 成功。
- `docker build` 成功。
- 镜像中不包含 `.env`。
- 容器可以启动 Gin 服务。
- 你能解释为什么容器里的 `127.0.0.1` 不是宿主机。

下一节使用 Docker Compose 同时启动 App、MySQL 和 Redis。
