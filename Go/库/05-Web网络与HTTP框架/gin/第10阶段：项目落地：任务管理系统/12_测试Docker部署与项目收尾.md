# 12. 测试、Docker 部署与项目收尾

本节目标：为任务管理系统补上测试、Docker Compose 和 README，让项目真正完整。

---

## 一、最低测试清单

优先测试关键路径：

```text
注册成功
邮箱重复失败
登录成功
登录密码错误失败
未登录访问任务失败
创建任务成功
查询别人任务失败
非法任务状态失败
```

不要一开始追求 100% 覆盖率。先保护最重要的业务行为。

---

## 二、Makefile

```makefile
.PHONY: run test fmt vet tidy check

run:
	go run ./cmd/server

test:
	go test ./...

fmt:
	gofmt -w .

vet:
	go vet ./...

tidy:
	go mod tidy

check: fmt vet test
```

提交代码前运行：

```bash
make check
```

Windows 如果没有 make，可以直接执行对应 Go 命令。

---

## 三、Dockerfile

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

---

## 四、docker-compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      GIN_MODE: release
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: task_api
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

注意：容器内 app 连接 MySQL 时 DSN 应该使用：

```text
mysql:3306
```

不是：

```text
127.0.0.1:3306
```

---

## 五、部署验证

启动：

```bash
docker compose up -d
```

查看：

```bash
docker compose ps
docker compose logs -f app
```

验证：

```http
GET http://localhost:8080/healthz
```

然后按顺序测试：

1. 注册。
2. 登录。
3. 携带 token 创建任务。
4. 查询任务列表。
5. 更新任务状态。
6. 删除任务。

---

## 六、README 必须写什么

项目 README 至少包含：

- 项目简介。
- 技术栈。
- 目录结构。
- 配置说明。
- 本地运行方式。
- Docker 运行方式。
- 测试命令。
- 核心接口示例。
- 常见问题。

别人能不能跑起来，是判断项目质量的重要标准。

---

## 七、项目复盘

完成后回答：

- 哪些逻辑放在 handler 里太多了？
- 哪些错误处理可以统一？
- 哪些 service 需要测试？
- 哪些接口缺少边界条件？
- 如果重写一次，目录结构会怎么调整？

复盘比“写完”更重要。

---

## 八、最终验收

项目完成时你应该能够：

- 从 0 创建 Gin 项目。
- 设计 RESTful API。
- 使用 GORM 操作数据库。
- 实现 JWT 登录认证。
- 编写认证中间件。
- 使用 Redis 做缓存和限流。
- 编写测试和 Makefile。
- 使用 Docker Compose 部署。
- 写出清晰 README。

做到这里，Gin 入门阶段就不只是看过教程，而是完成了一个可展示的后端项目。

---

## 九、测试矩阵

项目收尾时，至少准备下面这些测试。

service 测试：

```text
注册成功
邮箱重复
登录成功
密码错误
创建任务成功
查询别人任务失败
更新任务状态成功
非法状态失败
```

handler 测试：

```text
注册参数错误
登录参数错误
未登录访问 profile
携带 token 访问 profile
创建任务参数错误
查询任务不存在
```

运行：

```bash
go test ./...
go test ./... -cover
```

学习项目不必一开始追求非常高覆盖率，但核心业务和权限边界必须测。

---

## 十、Docker Compose 完整检查

启动：

```bash
docker compose up -d --build
```

查看状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f app
docker compose logs -f mysql
docker compose logs -f redis
```

验收：

- [ ] app 正常启动。
- [ ] mysql 正常启动。
- [ ] redis 正常启动。
- [ ] app 能连接 mysql。
- [ ] app 能连接 redis。
- [ ] `/healthz` 返回 200。
- [ ] `/readyz` 返回 200。

如果 app 连不上 MySQL，优先检查 DSN 是否使用 `mysql:3306`。

---

## 十一、上线前请求回归

按顺序执行：

```http
### health
GET http://localhost:8080/healthz

### register
POST http://localhost:8080/api/v1/auth/register
Content-Type: application/json

{
  "email": "demo@example.com",
  "password": "123456",
  "nickname": "demo"
}

### login
POST http://localhost:8080/api/v1/auth/login
Content-Type: application/json

{
  "email": "demo@example.com",
  "password": "123456"
}

### create task
POST http://localhost:8080/api/v1/tasks
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "title": "完成 Gin 项目",
  "description": "测试完整链路",
  "status": "todo"
}
```

后续继续测列表、详情、更新、删除。每个请求都记录状态码和业务 code。

---

## 十二、README 收尾模板

README 中建议加入：

````md
# Task API

基于 Gin、GORM、JWT、Redis 的任务管理系统后端 API。

## 功能

- 注册登录
- JWT 认证
- 用户资料
- 任务 CRUD
- 登录失败限制
- Docker 部署

## 本地启动

```bash
go mod tidy
go run ./cmd/server -config=config/config.local.yaml
```

## Docker 启动

```bash
cp .env.example .env
docker compose up -d --build
```

## 测试

```bash
go test ./...
```

## 健康检查

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```
````

README 不是装饰，它是别人运行你项目的入口。

---

## 十三、项目收尾排错

### 1. 测试依赖顺序

如果测试必须按固定顺序运行，说明测试设计不够独立。尽量让每个测试自己准备数据。

### 2. Docker 启动成功但接口失败

查看：

```bash
docker compose logs app
```

再检查 `/readyz`。

### 3. 本地能跑，Docker 不能跑

大概率是配置问题。容器里不能用本机视角的 `127.0.0.1` 访问 MySQL 或 Redis。

### 4. README 跟实际命令不一致

按 README 从零跑一遍。如果跑不通，README 就不合格。

---

## 十四、最终交付清单

提交项：

- [ ] 源码结构清晰。
- [ ] 配置文件有示例。
- [ ] `.env.example` 存在。
- [ ] `.env` 不提交。
- [ ] 测试能运行。
- [ ] Docker Compose 能启动。
- [ ] README 完整。
- [ ] requests.http 或接口文档完整。
- [ ] 项目复盘已记录。

完成这些，第10阶段才算真正收尾。
