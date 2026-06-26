# 1. 使用 Docker 启动 PostgreSQL

本节目标：使用 Docker 在本地启动一个 PostgreSQL 数据库，掌握容器启动、连接、停止、重启、数据持久化和 Docker Compose 管理方式。

推荐初学阶段使用 Docker 启动 PostgreSQL，原因是它更容易隔离环境、重置数据和切换版本。相比直接安装到操作系统里，Docker 更适合反复练习和快速清理。

---

## 一、前置条件

开始之前，需要确认：

- 已安装 Docker Desktop。
- Docker Desktop 正在运行。
- 终端中可以执行 `docker --version`。
- 本机 `5432` 端口没有被其他 PostgreSQL 服务占用。

如果你已经在本机安装过 PostgreSQL，可能会占用 `5432` 端口。遇到端口冲突时，可以把宿主机端口改成 `5433`，后文会给出示例。

---

## 二、最小启动命令

第一次体验 PostgreSQL，可以先使用下面的命令启动一个容器：

```bash
docker run --name my-postgres `
  -e POSTGRES_PASSWORD=mysecretpassword `
  -p 5432:5432 `
  -d postgres:16
```

如果你使用的是 Linux/macOS 终端，可以写成：

```bash
docker run --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  -d postgres:16
```

参数说明：

- `docker run`：创建并启动一个新容器。
- `--name my-postgres`：给容器指定名称，方便后续管理。
- `-e POSTGRES_PASSWORD=mysecretpassword`：设置默认超级用户 `postgres` 的密码。
- `-p 5432:5432`：把本机的 `5432` 端口映射到容器内的 `5432` 端口。
- `-d`：让容器在后台运行。
- `postgres:16`：使用 PostgreSQL 16 镜像。

注意：`mysecretpassword` 只适合本地学习，不要在真实项目或生产环境中使用这种简单密码。

---

## 三、检查容器是否启动成功

查看正在运行的容器：

```bash
docker ps
```

如果能看到 `my-postgres`，说明容器已经启动。

查看 PostgreSQL 容器日志：

```bash
docker logs my-postgres
```

如果日志中出现类似 `database system is ready to accept connections` 的信息，说明数据库已经可以连接。

---

## 四、连接 PostgreSQL

### 方式一：使用图形化客户端连接

可以使用 DBeaver、DataGrip、Navicat 或 pgAdmin 连接。

连接参数：

```text
Host: localhost
Port: 5432
Database: postgres
User: postgres
Password: mysecretpassword
```

如果你把宿主机端口改成了 `5433`，这里的 `Port` 也要改成 `5433`。

### 方式二：进入容器使用 psql

不安装额外客户端，也可以直接进入容器内使用 PostgreSQL 自带的 `psql`：

```bash
docker exec -it my-postgres psql -U postgres
```

进入后可以执行：

```sql
SELECT version();
```

退出 `psql`：

```text
\q
```

---

## 五、创建学习用数据库

连接成功后，建议不要一直使用默认的 `postgres` 数据库。可以创建一个专门用于学习的数据库：

```sql
CREATE DATABASE learn_pg;
```

查看数据库列表：

```sql
\l
```

如果使用图形化客户端，也可以在客户端里新建数据库。

---

## 六、重要：数据持久化

前面的最小启动命令适合临时体验，但不适合作为长期学习环境。

如果容器被删除，容器内部的数据也会丢失。为了让数据保存在 Docker volume 中，建议使用下面这个版本。

PowerShell 写法：

```bash
docker stop my-postgres
docker rm my-postgres

docker run --name my-postgres `
  -e POSTGRES_PASSWORD=mysecretpassword `
  -p 5432:5432 `
  -v pgdata:/var/lib/postgresql/data `
  -d postgres:16
```

Linux/macOS 写法：

```bash
docker stop my-postgres
docker rm my-postgres

docker run --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres:16
```

新增参数说明：

- `-v pgdata:/var/lib/postgresql/data`：创建并挂载一个名为 `pgdata` 的 Docker volume。
- `/var/lib/postgresql/data`：PostgreSQL 在容器内默认保存数据的位置。

这样即使删除容器，只要没有删除 `pgdata` 这个 volume，数据库数据仍然会保留。

查看 volume：

```bash
docker volume ls
```

注意：如果执行了删除 volume 的操作，数据库数据也会被删除。

---

## 七、日常管理命令

停止数据库：

```bash
docker stop my-postgres
```

重新启动数据库：

```bash
docker start my-postgres
```

查看正在运行的容器：

```bash
docker ps
```

查看所有容器，包括已经停止的容器：

```bash
docker ps -a
```

查看日志：

```bash
docker logs my-postgres
```

删除容器：

```bash
docker rm my-postgres
```

删除容器前需要先停止容器。删除容器不等于删除 volume，除非你额外删除了对应的数据卷。

---

## 八、端口冲突处理

如果启动时提示 `5432` 端口已被占用，可以把本机端口改成 `5433`：

```bash
docker run --name my-postgres `
  -e POSTGRES_PASSWORD=mysecretpassword `
  -p 5433:5432 `
  -v pgdata:/var/lib/postgresql/data `
  -d postgres:16
```

此时连接参数变为：

```text
Host: localhost
Port: 5433
Database: postgres
User: postgres
Password: mysecretpassword
```

注意：`5433:5432` 的含义是：

- 左边的 `5433` 是本机端口。
- 右边的 `5432` 是容器内部 PostgreSQL 端口。

---

## 九、推荐方案：使用 Docker Compose

熟悉基本命令后，推荐使用 Docker Compose 管理本地 PostgreSQL。它可以把启动参数写进配置文件，避免每次手动输入很长的命令。

在当前学习目录中创建 `docker-compose.yml`：

```yaml
services:
  db:
    image: postgres:16
    container_name: local-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: mysecretpassword
      POSTGRES_DB: learn_pg
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

启动：

```bash
docker compose up -d
```

查看运行状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs db
```

停止并移除容器：

```bash
docker compose down
```

注意：`docker compose down` 默认不会删除 volume，所以数据库数据仍然保留。

如果执行下面的命令，会连同 volume 一起删除，数据库数据也会丢失：

```bash
docker compose down -v
```

执行带 `-v` 的命令前一定要确认是否真的不再需要这些数据。

---

## 十、常见问题

### 1. 连接不上数据库

可以依次检查：

- Docker Desktop 是否正在运行。
- 容器是否启动：`docker ps`。
- 端口是否写对。
- 密码是否写对。
- 日志中是否出现启动失败信息：`docker logs my-postgres`。

### 2. 端口 5432 被占用

把端口映射改成：

```text
5433:5432
```

然后客户端连接 `localhost:5433`。

### 3. 删除容器后数据不见了

可能是最初启动时没有挂载 volume，或者删除了 volume。

长期学习建议始终使用：

```bash
-v pgdata:/var/lib/postgresql/data
```

或者使用 Docker Compose 中的：

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

### 4. 修改了 POSTGRES_PASSWORD 但密码没有变化

PostgreSQL 镜像只会在第一次初始化数据目录时读取 `POSTGRES_PASSWORD`。

如果 volume 已经存在，后续修改环境变量不会自动修改已有数据库用户的密码。需要进入数据库执行 `ALTER USER`，或者删除旧 volume 后重新初始化。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 使用 Docker 启动 PostgreSQL。
- 使用图形化客户端或 `psql` 连接数据库。
- 创建一个学习用数据库。
- 理解端口映射 `5432:5432` 的含义。
- 理解为什么需要 volume 做数据持久化。
- 能停止、启动、查看日志和删除 PostgreSQL 容器。
- 能使用 Docker Compose 管理本地 PostgreSQL。

掌握这些之后，就可以进入下一节：连接 PostgreSQL 并开始编写第一批 SQL。