# 04 完整本地练习环境搭建

本节目标：搭建一个长期可复用的 GORM 学习环境。学完后，你不只是“装好了数据库”，而是能启动数据库、连接数据库、创建学习库、准备 Go 项目、验证 GORM 能正常访问数据库。

前面几节已经讲了 Go、SQL 和学习方法。本节把这些内容串起来，做成一个可以反复使用的本地实验室。

---

## 一、为什么要先搭实验室

很多初学者学 GORM 会卡在这些地方：

- Go 代码没问题，但数据库没启动。
- 数据库连接字符串写错。
- 数据库名不存在。
- 字符集不支持中文。
- 时间字段扫描失败。
- 表结构改了，但自己不知道数据库里实际是什么样。

所以正式写 GORM 之前，先把环境固定下来非常重要。后续每个阶段都默认你有一个可用的数据库和一个可运行的 Go 项目。

---

## 二、推荐目录

建议在当前 GORM 学习目录下新建一个代码练习项目：

```text
GORM
├── gorm-learning-roadmap.md
├── 00-preparation
├── 01-gorm-quickstart
└── code
    └── gorm-study
        ├── go.mod
        └── main.go
```

命令：

```powershell
mkdir code
cd code
mkdir gorm-study
cd gorm-study
go mod init gorm-study
```

如果你已经有自己的练习项目，也可以直接使用自己的项目。关键是：教程文档和练习代码分开放，后续查找会更清晰。

---

## 三、使用 Docker 启动 MySQL

如果你本机已经安装 MySQL，可以跳过这一节。为了学习环境稳定，推荐用 Docker。

PowerShell 命令：

```powershell
docker run --name gorm-mysql `
  -e MYSQL_ROOT_PASSWORD=root123456 `
  -e MYSQL_DATABASE=gorm_study `
  -p 3306:3306 `
  -v gorm_mysql_data:/var/lib/mysql `
  -d mysql:8.4
```

参数说明：

- `--name gorm-mysql`：容器名，后续管理时会用到。
- `MYSQL_ROOT_PASSWORD=root123456`：root 用户密码。
- `MYSQL_DATABASE=gorm_study`：容器首次初始化时自动创建学习数据库。
- `-p 3306:3306`：把本机 3306 映射到容器 3306。
- `-v gorm_mysql_data:/var/lib/mysql`：使用 volume 保存数据。
- `mysql:8.4`：使用 MySQL 8.4 镜像。

查看容器：

```powershell
docker ps
```

查看日志：

```powershell
docker logs gorm-mysql
```

如果看到数据库已经 ready，说明可以连接。

---

## 四、端口冲突处理

如果你的本机已经有 MySQL，占用了 `3306`，可以改成本机 `3307`：

```powershell
docker run --name gorm-mysql `
  -e MYSQL_ROOT_PASSWORD=root123456 `
  -e MYSQL_DATABASE=gorm_study `
  -p 3307:3306 `
  -v gorm_mysql_data:/var/lib/mysql `
  -d mysql:8.4
```

此时连接参数变为：

```text
Host: 127.0.0.1
Port: 3307
User: root
Password: root123456
Database: gorm_study
```

注意 `3307:3306` 的含义：

- 左边 `3307` 是你电脑上的端口。
- 右边 `3306` 是容器内部 MySQL 的端口。

---

## 五、使用 Docker Compose 管理数据库

长期学习更推荐 Docker Compose。创建：

```text
code/gorm-study/docker-compose.yml
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

启动：

```powershell
docker compose up -d
```

停止：

```powershell
docker compose down
```

注意：`docker compose down` 默认不会删除 volume。不要轻易执行：

```powershell
docker compose down -v
```

带 `-v` 会删除数据库数据。

---

## 六、连接数据库

用 DBeaver、DataGrip 或 Navicat 连接：

```text
Host: 127.0.0.1
Port: 3306
User: root
Password: root123456
Database: gorm_study
```

连接成功后执行：

```sql
SELECT VERSION();
```

再执行：

```sql
SHOW DATABASES;
```

确认能看到：

```text
gorm_study
```

---

## 七、准备 Go 连接测试

进入 `code/gorm-study`，安装依赖：

```powershell
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

创建 `main.go`：

```go
package main

import (
    "fmt"
    "log"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

func main() {
    dsn := "root:root123456@tcp(127.0.0.1:3306)/gorm_study?charset=utf8mb4&parseTime=True&loc=Local"

    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        log.Fatal(err)
    }

    if err := sqlDB.Ping(); err != nil {
        log.Fatal(err)
    }

    fmt.Println("gorm mysql connected")
}
```

运行：

```powershell
go run .
```

预期输出：

```text
gorm mysql connected
```

---

## 八、DSN 逐段解释

```text
root:root123456@tcp(127.0.0.1:3306)/gorm_study?charset=utf8mb4&parseTime=True&loc=Local
```

拆开看：

```text
root             用户名
root123456       密码
tcp              使用 TCP 连接
127.0.0.1:3306   数据库地址和端口
gorm_study       数据库名
charset=utf8mb4  支持中文和 emoji 等完整字符
parseTime=True   把数据库时间解析为 Go time.Time
loc=Local        使用本地时区
```

其中 `parseTime=True` 很重要。如果不加，后面使用 `time.Time` 字段时容易遇到扫描问题。

---

## 九、常见错误

### 1. Access denied

类似：

```text
Access denied for user 'root'
```

通常是密码不对。检查 DSN 中的密码是否和 `MYSQL_ROOT_PASSWORD` 一致。

### 2. Unknown database

类似：

```text
Unknown database 'gorm_study'
```

说明数据库不存在。可以手动创建：

```sql
CREATE DATABASE gorm_study DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 3. connection refused

通常是数据库没启动或端口不对。

检查：

```powershell
docker ps
docker logs gorm-mysql
```

### 4. 中文乱码

确认建库字符集：

```sql
SHOW CREATE DATABASE gorm_study;
```

建议包含：

```text
utf8mb4
```

DSN 也要包含：

```text
charset=utf8mb4
```

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 使用 Docker 或 Docker Compose 启动 MySQL。
- 理解端口映射和 volume 的作用。
- 使用图形化客户端连接 `gorm_study` 数据库。
- 创建一个 Go Module 项目。
- 安装 GORM 和 MySQL 驱动。
- 写出最小 GORM 连接测试程序。
- 能解释 DSN 中每个参数的含义。
- 遇到连接失败时，能按日志、端口、账号、数据库名逐项排查。
