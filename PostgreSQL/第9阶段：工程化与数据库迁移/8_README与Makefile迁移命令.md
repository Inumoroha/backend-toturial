# 8. README 与 Makefile 迁移命令

本节目标：把迁移命令写进项目 README 或 Makefile，让团队成员能用统一方式执行迁移，而不是每个人记一套命令。

工具会用还不够。

真实项目里还要做到：

```text
新同事能看 README 跑起来。
CI 能执行迁移。
团队使用同一套命令。
命令参数清楚。
不会误操作生产。
```

---

## 一、为什么要写 README

如果没有文档，新同事会问：

```text
数据库怎么建？
迁移怎么跑？
seed 怎么导入？
回滚怎么操作？
DATABASE_URL 格式是什么？
用 goose 还是 migrate？
```

README 至少应该写清楚：

- 使用哪个迁移工具。
- 如何安装工具。
- 环境变量怎么设置。
- 如何执行迁移。
- 如何回滚一步。
- 如何查看状态。
- 如何导入本地 seed。

---

## 二、README 示例：goose

````md
# Database

## Environment

Set database URL:

```powershell
$env:DATABASE_URL="postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable"
```

## Install goose

```powershell
go install github.com/pressly/goose/v3/cmd/goose@latest
```

## Run migrations

```powershell
goose -dir migrations postgres $env:DATABASE_URL up
```

## Rollback one migration

```powershell
goose -dir migrations postgres $env:DATABASE_URL down
```

## Migration status

```powershell
goose -dir migrations postgres $env:DATABASE_URL status
```

## Create migration

```powershell
goose -dir migrations -s create add_users_avatar_url sql
```

## Load local seed

```powershell
psql $env:DATABASE_URL -f seeds/local_seed.sql
```
````

注意：上面是 README 内容示例，不要把生产连接字符串写进 README。

---

## 三、README 示例：golang-migrate

````md
# Database

## Environment

```powershell
$env:DATABASE_URL="postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable"
```

## Install migrate

```powershell
go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

## Run migrations

```powershell
migrate -path migrations -database $env:DATABASE_URL up
```

## Rollback one migration

```powershell
migrate -path migrations -database $env:DATABASE_URL down 1
```

## Current version

```powershell
migrate -path migrations -database $env:DATABASE_URL version
```

## Create migration

```powershell
migrate create -ext sql -dir migrations -seq add_users_avatar_url
```
````

---

## 四、为什么要写 Makefile

README 是给人看的。

Makefile 是给人和 CI 执行的。

有了 Makefile，命令可以变短：

```powershell
make migrate-up
make migrate-down
make migrate-status
```

不过 Windows 默认不一定有 `make`。如果你的团队主要用 Windows，可以考虑：

- 使用 PowerShell 脚本。
- 使用 Taskfile。
- 使用 just。
- 在 README 中保留完整命令。

本教程给出 Makefile 主要是展示工程化思路。

---

## 五、Makefile 示例：goose

```makefile
MIGRATION_DIR := migrations
DB_DRIVER := postgres

.PHONY: migrate-up
migrate-up:
	goose -dir $(MIGRATION_DIR) $(DB_DRIVER) "$(DATABASE_URL)" up

.PHONY: migrate-down
migrate-down:
	goose -dir $(MIGRATION_DIR) $(DB_DRIVER) "$(DATABASE_URL)" down

.PHONY: migrate-status
migrate-status:
	goose -dir $(MIGRATION_DIR) $(DB_DRIVER) "$(DATABASE_URL)" status

.PHONY: migrate-create
migrate-create:
	goose -dir $(MIGRATION_DIR) -s create $(name) sql

.PHONY: seed-local
seed-local:
	psql "$(DATABASE_URL)" -f seeds/local_seed.sql
```

使用：

```powershell
$env:DATABASE_URL="postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable"
make migrate-up
make migrate-create name=add_users_avatar_url
```

---

## 六、Makefile 示例：golang-migrate

```makefile
MIGRATION_DIR := migrations

.PHONY: migrate-up
migrate-up:
	migrate -path $(MIGRATION_DIR) -database "$(DATABASE_URL)" up

.PHONY: migrate-down
migrate-down:
	migrate -path $(MIGRATION_DIR) -database "$(DATABASE_URL)" down 1

.PHONY: migrate-version
migrate-version:
	migrate -path $(MIGRATION_DIR) -database "$(DATABASE_URL)" version

.PHONY: migrate-create
migrate-create:
	migrate create -ext sql -dir $(MIGRATION_DIR) -seq $(name)
```

使用：

```powershell
make migrate-up
make migrate-down
make migrate-create name=add_users_avatar_url
```

---

## 七、防止误操作生产

Makefile 不应该默认连接生产数据库。

推荐：

```text
本地通过 DATABASE_URL 指定。
生产由 CI/CD 注入 secret。
生产迁移需要审批。
```

可以在脚本中加保护，例如检测环境变量：

```makefile
.PHONY: check-db-url
check-db-url:
	@if [ -z "$(DATABASE_URL)" ]; then echo "DATABASE_URL is required"; exit 1; fi
```

Windows PowerShell 语法不同，团队要根据实际环境调整。

---

## 八、CI 中执行迁移

CI 流程通常类似：

```text
启动 PostgreSQL 服务。
设置 DATABASE_URL。
执行 migrate-up。
运行 go test。
```

伪流程：

```yaml
steps:
  - checkout
  - setup-go
  - start-postgres
  - run: make migrate-up
  - run: go test ./...
```

CI 应该使用测试数据库，不要连接生产数据库。

---

## 九、团队约定要写清楚

README 里建议说明：

```text
一个项目只使用一种迁移工具。
迁移文件一旦合并，不要修改旧文件。
新增结构变化时，新建迁移。
生产迁移通过发布流程执行。
本地 seed 不能用于生产。
```

这些约定能减少很多后期事故。

---

## 十、常见误区

### 1. 会命令就不用 README？

不是。

项目要让别人也能稳定运行，而不是只靠你记得。

### 2. Makefile 里可以写死 DATABASE_URL？

不建议。

尤其不能写死生产连接字符串。

### 3. Windows 项目不能用工程化命令？

可以。

如果不用 Makefile，可以写 PowerShell 脚本或 README 完整命令。

### 4. CI 测试可以跳过迁移？

不建议。

CI 执行迁移能尽早发现迁移文件错误。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 在 README 中写清迁移工具安装和使用命令。
- 写出 goose 的迁移命令文档。
- 写出 golang-migrate 的迁移命令文档。
- 用 Makefile 封装迁移命令。
- 知道生产连接字符串不能写死。
- 知道 CI 应该执行迁移后再运行测试。
- 能写出团队迁移约定。
