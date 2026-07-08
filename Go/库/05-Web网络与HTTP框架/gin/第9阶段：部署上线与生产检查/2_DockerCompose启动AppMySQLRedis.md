# 2. Docker Compose 启动 App、MySQL、Redis

本节目标：使用 Docker Compose 一次性启动 Gin 应用、MySQL 和 Redis，并让它们通过服务名互相访问。

单独运行 app 容器只能验证镜像能启动。真实项目还需要数据库、Redis、Nginx 等依赖。Docker Compose 用来管理这些多容器服务。

---

## 一、Compose 中的服务名

在 Docker Compose 网络里，每个 service 名称都可以当作主机名使用。

例如：

```yaml
services:
  mysql:
    image: mysql:8

  redis:
    image: redis:7

  app:
    build: .
```

app 容器中可以访问：

```text
mysql:3306
redis:6379
```

不要写：

```text
127.0.0.1:3306
127.0.0.1:6379
```

---

## 二、编写 docker-compose.yml

项目根目录创建 `docker-compose.yml`：

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: gin_mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 3s
      retries: 20

  redis:
    image: redis:7
    container_name: gin_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: gin_app
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      APP_SERVER_PORT: 8080
      APP_DATABASE_DSN: ${APP_DATABASE_DSN}
      APP_REDIS_ADDR: redis:6379
      APP_JWT_SECRET: ${APP_JWT_SECRET}
    ports:
      - "8080:8080"
    volumes:
      - uploads_data:/app/uploads

volumes:
  mysql_data:
  redis_data:
  uploads_data:
```

---

## 三、编写 .env.example

项目根目录创建 `.env.example`：

```dotenv
MYSQL_ROOT_PASSWORD=change-me
MYSQL_DATABASE=gin_user_api
APP_DATABASE_DSN=root:change-me@tcp(mysql:3306)/gin_user_api?charset=utf8mb4&parseTime=True&loc=Local
APP_JWT_SECRET=please-change-this-secret
```

实际使用时复制：

```bash
cp .env.example .env
```

Windows PowerShell：

```powershell
Copy-Item .env.example .env
```

然后修改 `.env` 中的真实值。

注意：`.env` 不应该提交到 Git。

---

## 四、启动服务

执行：

```bash
docker compose up -d --build
```

查看服务：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f app
```

访问：

```bash
curl http://localhost:8080/ping
```

---

## 五、depends_on 的含义

配置中：

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

表示 app 等 mysql healthcheck 通过后再启动。

但要注意：这只能减少启动顺序问题，不能替代应用内部的重试和错误处理。生产中应用连接数据库失败时，最好有清晰日志，必要时支持重试。

---

## 六、数据持久化

Compose 中定义了 volumes：

```yaml
volumes:
  mysql_data:
  redis_data:
  uploads_data:
```

作用：

- `mysql_data`：保存 MySQL 数据。
- `redis_data`：保存 Redis 数据。
- `uploads_data`：保存上传文件。

没有 volume 时，容器删除后数据可能一起丢失。

---

## 七、进入容器排查

进入 app 容器：

```bash
docker compose exec app sh
```

在容器里检查环境变量：

```sh
env | grep APP
```

检查网络：

```sh
ping mysql
ping redis
```

Alpine 镜像可能没有 ping，可以临时安装或通过应用日志判断。

进入 MySQL：

```bash
docker compose exec mysql mysql -uroot -p
```

进入 Redis：

```bash
docker compose exec redis redis-cli ping
```

---

## 八、常见问题

### 1. app 连不上 MySQL

检查 DSN 是否使用了 `mysql:3306`，而不是 `127.0.0.1:3306`。

### 2. app 启动早于数据库

确认 MySQL healthcheck 是否通过，查看：

```bash
docker compose ps
docker compose logs mysql
```

### 3. 修改代码后容器没有更新

重新构建：

```bash
docker compose up -d --build
```

### 4. 端口被占用

如果本机已有 MySQL 占用 3306，可以改端口映射：

```yaml
ports:
  - "13306:3306"
```

容器内部 app 仍然访问 `mysql:3306`。

---

## 九、练习

请完成：

1. 编写 `docker-compose.yml`。
2. 编写 `.env.example`。
3. 复制 `.env` 并修改值。
4. 启动 app、mysql、redis。
5. 通过 app 容器连接 mysql 和 redis。
6. 停止并重启服务，确认数据仍在。

---

## 十、验收标准

完成后确认：

- `docker compose up -d --build` 成功。
- `docker compose ps` 中服务状态正常。
- app 日志中没有数据库或 Redis 连接错误。
- `/ping` 可访问。
- MySQL、Redis、uploads 都使用了 volume。

下一节会加入 Nginx、环境变量细节和部署排错思路。
