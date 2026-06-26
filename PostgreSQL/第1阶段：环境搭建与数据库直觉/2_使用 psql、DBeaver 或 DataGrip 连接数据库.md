# 2. 使用 psql、DBeaver 或 DataGrip 连接数据库

本节目标：在 PostgreSQL 容器启动后，使用命令行客户端 `psql`、图形化客户端 DBeaver 或 DataGrip 连接数据库，并能执行简单 SQL 验证连接是否成功。

连接数据库是后续学习 SQL、表设计和 Go 项目接入 PostgreSQL 的前置步骤。无论使用哪种客户端，本质上都需要填写同一组连接参数。

---

## 一、连接前确认

开始连接之前，先确认 PostgreSQL 容器正在运行。

查看正在运行的容器：

```bash
docker ps
```

如果没有看到 PostgreSQL 容器，可以启动它。

如果你使用的是第 1 节中的 `docker run` 方式：

```bash
docker start my-postgres
```

如果你使用的是 Docker Compose：

```bash
docker compose up -d
```

查看日志：

```bash
docker logs my-postgres
```

如果日志中出现类似 `database system is ready to accept connections` 的信息，说明数据库已经可以连接。

---

## 二、连接参数说明

以第 1 节的本地 Docker 环境为例，常用连接参数如下：

```text
Host: localhost
Port: 5432
Database: learn_pg
User: postgres
Password: mysecretpassword
```

参数含义：

- `Host`：数据库所在主机。本地 Docker 端口已经映射到本机，所以使用 `localhost` 或 `127.0.0.1`。
- `Port`：数据库端口。PostgreSQL 默认端口是 `5432`。
- `Database`：要连接的数据库名。推荐使用学习数据库 `learn_pg`。
- `User`：数据库用户名。默认超级用户是 `postgres`。
- `Password`：启动容器时设置的密码。

如果你还没有创建 `learn_pg` 数据库，可以先连接默认数据库 `postgres`，然后执行：

```sql
CREATE DATABASE learn_pg;
```

如果第 1 节中因为端口冲突把端口改成了 `5433:5432`，客户端连接时要使用：

```text
Port: 5433
```

---

## 三、使用 psql 连接数据库

`psql` 是 PostgreSQL 官方命令行客户端。它适合快速执行 SQL、查看数据库结构，也适合后续学习 PostgreSQL 的基础命令。

### 方式一：进入容器内使用 psql

如果本机没有安装 `psql`，可以直接使用容器内自带的 `psql`。

连接默认数据库：

```bash
docker exec -it my-postgres psql -U postgres
```

连接指定数据库 `learn_pg`：

```bash
docker exec -it my-postgres psql -U postgres -d learn_pg
```

进入后，命令行提示符通常会变成类似下面的形式：

```text
learn_pg=#
```

这说明你已经进入 PostgreSQL 交互环境。

### 方式二：使用本机安装的 psql 连接

如果本机已经安装了 PostgreSQL 客户端工具，也可以从宿主机直接连接：

```bash
psql -h localhost -p 5432 -U postgres -d learn_pg
```

如果端口映射使用的是 `5433:5432`，命令改成：

```bash
psql -h localhost -p 5433 -U postgres -d learn_pg
```

执行后会提示输入密码，输入第 1 节中设置的密码即可。

### 常用 psql 命令

进入 `psql` 后，可以使用这些常用命令：

```text
\l          查看所有数据库
\c dbname   切换到指定数据库
\dt         查看当前数据库中的表
\d table    查看指定表结构
\conninfo   查看当前连接信息
\q          退出 psql
```

注意：这些是 `psql` 的元命令，不是标准 SQL。它们通常以反斜杠 `\` 开头。

---

## 四、使用 DBeaver 连接数据库

DBeaver 是常用的免费图形化数据库客户端，适合初学者查看表结构、浏览数据和执行 SQL。

操作步骤：

1. 打开 DBeaver。
2. 新建数据库连接。
3. 数据库类型选择 PostgreSQL。
4. 填写连接参数：

```text
Host: localhost
Port: 5432
Database: learn_pg
Username: postgres
Password: mysecretpassword
```

5. 点击 `Test Connection` 测试连接。
6. 如果提示缺少驱动，按照 DBeaver 提示下载 PostgreSQL 驱动。
7. 测试成功后保存连接。

连接成功后，可以在左侧导航中查看数据库、schema、表和视图。

如果你还没有创建 `learn_pg`，可以先把 `Database` 填成 `postgres`，连接成功后执行：

```sql
CREATE DATABASE learn_pg;
```

---

## 五、使用 DataGrip 连接数据库

DataGrip 是 JetBrains 出品的专业数据库客户端，适合长期开发使用。它功能较强，但通常需要付费或使用授权。

操作步骤：

1. 打开 DataGrip。
2. 在 Database 面板中新建数据源。
3. 选择 PostgreSQL。
4. 填写连接参数：

```text
Host: localhost
Port: 5432
Database: learn_pg
User: postgres
Password: mysecretpassword
```

5. 如果底部提示缺少驱动，点击 `Download` 下载 PostgreSQL 驱动。
6. 点击 `Test Connection`。
7. 测试成功后保存连接。

连接成功后，可以打开 Console 编写并执行 SQL。

---

## 六、连接成功后的验证 SQL

无论使用 `psql`、DBeaver 还是 DataGrip，连接成功后都建议执行下面几条 SQL，确认当前连接信息正确。

查看 PostgreSQL 版本：

```sql
SELECT version();
```

查看当前数据库：

```sql
SELECT current_database();
```

查看当前用户：

```sql
SELECT current_user;
```

查看当前时间：

```sql
SELECT now();
```

如果 `current_database()` 返回 `learn_pg`，说明你已经连接到了学习数据库。

---

## 七、常见问题

### 1. Connection refused

常见原因：

- PostgreSQL 容器没有启动。
- 端口填写错误。
- Docker 端口映射不是 `5432:5432`。

排查命令：

```bash
docker ps
docker logs my-postgres
```

如果使用了 `5433:5432`，客户端端口要填写 `5433`。

### 2. password authentication failed

常见原因：

- 密码输入错误。
- 使用了旧 volume，后来修改 `POSTGRES_PASSWORD` 没有生效。

PostgreSQL 镜像只会在第一次初始化数据目录时读取 `POSTGRES_PASSWORD`。如果 volume 已经存在，后续修改环境变量不会自动修改已有用户密码。

### 3. database "learn_pg" does not exist

说明当前 PostgreSQL 中还没有创建 `learn_pg` 数据库。

可以先连接默认数据库 `postgres`，然后执行：

```sql
CREATE DATABASE learn_pg;
```

### 4. DBeaver 或 DataGrip 提示缺少驱动

第一次连接 PostgreSQL 时，图形化客户端可能需要下载 PostgreSQL JDBC 驱动。

处理方式：

- DBeaver：按照提示下载驱动。
- DataGrip：点击 `Download missing driver files` 或类似按钮。

### 5. 连接到了 postgres，但不是 learn_pg

`postgres` 是默认数据库，能连接成功不代表已经连接到学习数据库。

可以执行：

```sql
SELECT current_database();
```

确认当前数据库名称。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 说清楚连接 PostgreSQL 需要哪些参数。
- 使用 `psql` 连接 PostgreSQL。
- 使用 DBeaver 或 DataGrip 连接 PostgreSQL。
- 执行 SQL 验证当前数据库、当前用户和 PostgreSQL 版本。
- 能处理端口错误、密码错误、数据库不存在、驱动缺失等常见问题。

掌握这些之后，就可以进入下一节：理解数据库、schema、table、row、column 等基本概念。