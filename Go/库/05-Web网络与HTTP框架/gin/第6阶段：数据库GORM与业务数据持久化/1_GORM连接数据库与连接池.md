# 1. GORM 连接数据库与连接池

本节目标：使用 GORM 连接 MySQL，并配置基础连接池。完成本节后，你应该能让 Gin 项目在启动时连接数据库，并能排查常见连接失败问题。

---

## 一、准备 MySQL

如果你本机已经有 MySQL，可以直接使用。

如果没有，建议用 Docker 启动一个学习环境：

```bash
docker run --name gin-mysql `
  -e MYSQL_ROOT_PASSWORD=password `
  -e MYSQL_DATABASE=gin_learning `
  -p 3306:3306 `
  -v gin_mysql_data:/var/lib/mysql `
  -d mysql:8.0
```

Linux/macOS：

```bash
docker run --name gin-mysql \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=gin_learning \
  -p 3306:3306 \
  -v gin_mysql_data:/var/lib/mysql \
  -d mysql:8.0
```

参数说明：

- `MYSQL_ROOT_PASSWORD=password`：root 密码。
- `MYSQL_DATABASE=gin_learning`：启动时创建学习数据库。
- `-p 3306:3306`：把本机 3306 映射到容器 3306。
- `-v gin_mysql_data:/var/lib/mysql`：持久化数据。

---

## 二、检查 MySQL 是否启动

```bash
docker ps
```

查看日志：

```bash
docker logs gin-mysql
```

如果看到类似：

```text
ready for connections
```

说明 MySQL 已经可以连接。

如果你使用图形化工具，连接参数：

```text
Host: 127.0.0.1
Port: 3306
User: root
Password: password
Database: gin_learning
```

---

## 三、安装 GORM

在 Go 项目根目录执行：

```bash
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

检查 `go.mod` 中应出现：

```text
gorm.io/gorm
gorm.io/driver/mysql
```

---

## 四、理解 DSN

MySQL DSN 示例：

```text
root:password@tcp(127.0.0.1:3306)/gin_learning?charset=utf8mb4&parseTime=True&loc=Local
```

拆开看：

```text
root
用户名。

password
密码。

127.0.0.1:3306
数据库地址和端口。

gin_learning
数据库名。

charset=utf8mb4
字符集，支持 emoji 等完整 Unicode。

parseTime=True
让时间字段能正确映射到 time.Time。

loc=Local
使用本地时区。
```

`parseTime=True` 很重要。否则时间字段可能扫描失败或表现异常。

---

## 五、创建 database 包

创建：

```text
internal/database/mysql.go
```

代码：

```go
package database

import (
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

func NewMySQL(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }

    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)

    return db, nil
}
```

---

## 六、连接池参数解释

```go
sqlDB.SetMaxIdleConns(10)
```

最大空闲连接数。空闲连接可以复用，减少频繁建立连接的成本。

```go
sqlDB.SetMaxOpenConns(100)
```

最大打开连接数。防止应用无限制创建数据库连接。

```go
sqlDB.SetConnMaxLifetime(time.Hour)
```

连接最大生命周期。避免连接存在太久导致各种网络或数据库侧问题。

学习阶段不必过度调优，先知道它们控制连接池即可。

---

## 七、在 main 中连接数据库

示例：

```go
package main

import (
    "log"

    "gin-layer-demo/internal/database"
)

func main() {
    dsn := "root:password@tcp(127.0.0.1:3306)/gin_learning?charset=utf8mb4&parseTime=True&loc=Local"

    db, err := database.NewMySQL(dsn)
    if err != nil {
        log.Fatal(err)
    }

    log.Println("database connected", db != nil)
}
```

运行：

```bash
go run ./cmd/server
```

期望看到：

```text
database connected true
```

---

## 八、Ping 检查

可以在连接后做一次 Ping：

```go
sqlDB, err := db.DB()
if err != nil {
    log.Fatal(err)
}

if err := sqlDB.Ping(); err != nil {
    log.Fatal(err)
}
```

如果 Ping 成功，说明数据库连接确实可用。

---

## 九、本地和 Docker Compose 的地址区别

本地运行 Go 程序，连接本机 MySQL：

```text
127.0.0.1:3306
```

如果 Go 程序也在 Docker Compose 容器里，连接 MySQL 容器：

```text
mysql:3306
```

这是新手部署时最常踩的坑。

容器里的 `127.0.0.1` 指的是 app 容器自己，不是 MySQL 容器。

---

## 十、常见错误

### 1. access denied

现象：

```text
Access denied for user 'root'
```

检查：

- 用户名是否正确。
- 密码是否正确。
- Docker 环境变量是否和 DSN 一致。

### 2. unknown database

现象：

```text
Unknown database 'gin_learning'
```

说明数据库不存在。

解决：

```sql
CREATE DATABASE gin_learning DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 3. connection refused

检查：

- MySQL 是否启动。
- 端口是否映射。
- DSN host 和 port 是否正确。

### 4. 时间字段异常

检查 DSN 是否包含：

```text
parseTime=True&loc=Local
```

---

## 十一、本节练习

完成：

1. 使用 Docker 或本地 MySQL 准备 `gin_learning` 数据库。
2. 安装 GORM 和 MySQL driver。
3. 创建 `internal/database/mysql.go`。
4. 在 main 中调用 `database.NewMySQL`。
5. 使用 `Ping` 验证连接。
6. 故意写错密码，观察错误。
7. 故意写错数据库名，观察错误。

---

## 十二、本节验收

你应该能够回答：

- DSN 每一段分别表示什么？
- `parseTime=True` 有什么作用？
- 连接池的 `MaxIdleConns` 和 `MaxOpenConns` 分别控制什么？
- 本地运行和 Docker Compose 内运行时，数据库 host 有什么区别？
- `unknown database` 应该怎么排查？

