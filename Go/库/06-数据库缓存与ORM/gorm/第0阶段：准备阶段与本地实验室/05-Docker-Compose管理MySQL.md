# 05 使用 Docker Compose 管理 MySQL 学习环境

本节目标：把 MySQL 的启动参数固定到 `docker-compose.yml` 中，形成一个稳定、可重建、可长期使用的 GORM 学习数据库环境。

前面已经演示过 `docker run`。`docker run` 适合第一次体验，但命令长、参数多、容易忘。长期学习更推荐 Docker Compose，因为所有参数都写在文件里，后续只需要 `docker compose up -d`。

---

## 一、准备目录

建议在练习项目根目录创建：

```text
code
└── gorm-study
    ├── docker-compose.yml
    ├── go.mod
    └── main.go
```

如果还没有 `code/gorm-study`：

```powershell
mkdir code
cd code
mkdir gorm-study
cd gorm-study
go mod init gorm-study
```

---

## 二、编写 docker-compose.yml

创建：

```text
docker-compose.yml
```

内容：

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: gorm-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root123456
      MYSQL_DATABASE: gorm_study
      TZ: Asia/Shanghai
    ports:
      - "3306:3306"
    volumes:
      - gorm_mysql_data:/var/lib/mysql
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci

volumes:
  gorm_mysql_data:
```

---

## 三、逐项解释配置

### image

```yaml
image: mysql:8.4
```

指定 MySQL 镜像版本。学习阶段建议固定版本，不要写 `latest`，否则以后拉取到新版本时可能出现行为差异。

### container_name

```yaml
container_name: gorm-mysql
```

给容器起固定名字，方便查看日志、停止、重启。

### restart

```yaml
restart: unless-stopped
```

Docker 重启后，容器会自动启动。除非你手动停止它。

### environment

```yaml
MYSQL_ROOT_PASSWORD: root123456
MYSQL_DATABASE: gorm_study
TZ: Asia/Shanghai
```

含义：

- `MYSQL_ROOT_PASSWORD`：root 用户密码。
- `MYSQL_DATABASE`：首次初始化时创建的数据库。
- `TZ`：容器时区。

注意：这些环境变量只在数据目录首次初始化时生效。如果 volume 已经存在，修改密码不会自动改变已有数据库用户密码。

### ports

```yaml
3306:3306
```

左边是宿主机端口，右边是容器内部端口。如果本机 3306 被占用，可以写：

```yaml
3307:3306
```

然后连接数据库时端口使用 `3307`。

### volumes

```yaml
gorm_mysql_data:/var/lib/mysql
```

MySQL 数据保存在 Docker volume 中。删除容器不会删除数据，除非你删除 volume。

### command

```yaml
--character-set-server=utf8mb4
--collation-server=utf8mb4_unicode_ci
```

让 MySQL 默认使用 `utf8mb4` 字符集，避免中文、emoji 等字符出现问题。

---

## 四、启动和停止

启动：

```powershell
docker compose up -d
```

查看运行状态：

```powershell
docker compose ps
```

查看日志：

```powershell
docker compose logs mysql
```

停止：

```powershell
docker compose down
```

停止并删除数据：

```powershell
docker compose down -v
```

注意：`-v` 会删除 volume，学习数据会丢失。除非你明确想重置数据库，否则不要加。

---

## 五、验证数据库

进入容器：

```powershell
docker exec -it gorm-mysql mysql -uroot -proot123456
```

进入后执行：

```sql
SHOW DATABASES;
```

确认存在：

```text
gorm_study
```

查看字符集：

```sql
SHOW CREATE DATABASE gorm_study;
```

应该能看到 `utf8mb4`。

退出：

```sql
exit;
```

---

## 六、常见错误

### 1. 端口被占用

报错类似：

```text
Bind for 0.0.0.0:3306 failed
```

解决：把端口改为：

```yaml
ports:
  - "3307:3306"
```

然后 DSN 也改成：

```text
127.0.0.1:3307
```

### 2. 修改密码后不生效

如果 volume 已经初始化过，修改 `MYSQL_ROOT_PASSWORD` 不会改变旧密码。

解决方式：

- 用旧密码登录后执行 `ALTER USER`。
- 或者删除 volume 后重新初始化。

### 3. 容器一直重启

查看日志：

```powershell
docker compose logs mysql
```

常见原因是配置写错、端口冲突、数据目录权限问题。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 编写 MySQL 的 `docker-compose.yml`。
- 使用 `docker compose up -d` 启动数据库。
- 使用 `docker compose logs` 查看日志。
- 理解端口映射、volume、字符集配置。
- 在端口冲突时改用 `3307`。
- 知道 `docker compose down -v` 会删除数据。
