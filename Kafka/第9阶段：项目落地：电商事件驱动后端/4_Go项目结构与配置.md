# 4. Go 项目结构与配置

本节目标：搭建项目目录、配置结构和启动入口。

---

## 一、目录结构

```text
ecommerce-events/
  cmd/
    order-service/
    inventory-service/
    outbox-worker/
  internal/
    kafka/
    order/
    inventory/
    outbox/
    platform/
  deployments/
  migrations/
```

---

## 二、配置

```go
type Config struct {
    HTTPAddr string
    DatabaseURL string
    KafkaBrokers []string
    KafkaClientID string
    KafkaGroupID string
}
```

环境变量：

```text
HTTP_ADDR=:8080
DATABASE_URL=postgres://...
KAFKA_BROKERS=localhost:9092
```

---

## 三、main.go 职责

```text
加载配置
初始化 logger
初始化 db
初始化 kafka
组装 service
启动 HTTP 或 consumer
监听退出信号
```

main 不写业务逻辑。

---

## 四、internal/kafka

包含：

```text
producer.go
consumer.go
message.go
retry.go
dlq.go
```

---

## 五、验收

- `go test ./...` 能运行。
- 配置不硬编码。
- 每个 cmd 入口职责清晰。

---

## 六、初始化 Go Module

```powershell
mkdir ecommerce-events
cd ecommerce-events
go mod init ecommerce-events
```

建议依赖：

```powershell
go get github.com/twmb/franz-go/pkg/kgo
go get github.com/jackc/pgx/v5/pgxpool
```

如果暂时不接 Prometheus，可以后面再加。

---

## 七、创建目录

```powershell
New-Item -ItemType Directory -Force -Path cmd/order-service
New-Item -ItemType Directory -Force -Path cmd/inventory-service
New-Item -ItemType Directory -Force -Path cmd/outbox-worker
New-Item -ItemType Directory -Force -Path internal/kafka
New-Item -ItemType Directory -Force -Path internal/order
New-Item -ItemType Directory -Force -Path internal/inventory
New-Item -ItemType Directory -Force -Path internal/outbox
New-Item -ItemType Directory -Force -Path internal/platform/config
New-Item -ItemType Directory -Force -Path internal/platform/database
New-Item -ItemType Directory -Force -Path deployments
New-Item -ItemType Directory -Force -Path migrations
```

Linux/macOS 可以用：

```bash
mkdir -p cmd/{order-service,inventory-service,outbox-worker}
mkdir -p internal/{kafka,order,inventory,outbox}
mkdir -p internal/platform/{config,database}
mkdir -p deployments migrations
```

---

## 八、配置加载

`internal/platform/config/config.go`：

```go
package config

import (
    "os"
    "strings"
)

type Config struct {
    HTTPAddr     string
    DatabaseURL  string
    KafkaBrokers []string
    KafkaClientID string
    KafkaGroupID  string
}

func Load() Config {
    return Config{
        HTTPAddr:      getEnv("HTTP_ADDR", ":8080"),
        DatabaseURL:   os.Getenv("DATABASE_URL"),
        KafkaBrokers:  strings.Split(getEnv("KAFKA_BROKERS", "localhost:9092"), ","),
        KafkaClientID: getEnv("KAFKA_CLIENT_ID", "ecommerce-events"),
        KafkaGroupID:  os.Getenv("KAFKA_GROUP_ID"),
    }
}

func getEnv(key, fallback string) string {
    value := os.Getenv(key)
    if value == "" {
        return fallback
    }
    return value
}
```

---

## 九、数据库连接

`internal/platform/database/postgres.go`：

```go
package database

import (
    "context"

    "github.com/jackc/pgx/v5/pgxpool"
)

func Open(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
    return pgxpool.New(ctx, databaseURL)
}
```

main 中使用：

```go
pool, err := database.Open(ctx, cfg.DatabaseURL)
if err != nil {
    log.Fatal(err)
}
defer pool.Close()
```

---

## 十、main.go 模板

`cmd/order-service/main.go`：

```go
package main

import (
    "context"
    "log"
    "net/http"

    "ecommerce-events/internal/platform/config"
    "ecommerce-events/internal/platform/database"
)

func main() {
    ctx := context.Background()
    cfg := config.Load()

    pool, err := database.Open(ctx, cfg.DatabaseURL)
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close()

    mux := http.NewServeMux()

    log.Println("order-service listening on", cfg.HTTPAddr)
    if err := http.ListenAndServe(cfg.HTTPAddr, mux); err != nil {
        log.Fatal(err)
    }
}
```

后续章节会把 order handler 注册进去。

---

## 十一、配置示例

本地 `.env.example`：

```text
HTTP_ADDR=:8080
DATABASE_URL=postgres://postgres:postgres@localhost:5432/ecommerce?sslmode=disable
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=order-service
KAFKA_GROUP_ID=
```

不要提交真实密码。

---

## 十二、本节练习

1. 初始化 Go module。
2. 创建目录结构。
3. 实现配置加载。
4. 实现数据库连接。
5. 写出 `cmd/order-service/main.go`。
6. 运行 `go test ./...` 确认项目能编译。

---

## 十三、三个服务的配置差异

同一个项目里有三个入口，但它们需要的配置不完全一样。

| 服务 | 必需配置 |
| --- | --- |
| order-service | `HTTP_ADDR`、`DATABASE_URL` |
| outbox-worker | `DATABASE_URL`、`KAFKA_BROKERS`、`KAFKA_CLIENT_ID` |
| inventory-service | `DATABASE_URL`、`KAFKA_BROKERS`、`KAFKA_GROUP_ID` |

所以 `.env.example` 可以写全，但每个 `main.go` 只读取自己需要的部分。

示例：

```text
ORDER_HTTP_ADDR=:8080
DATABASE_URL=postgres://postgres:postgres@localhost:5432/ecommerce?sslmode=disable
KAFKA_BROKERS=localhost:9092
ORDER_KAFKA_CLIENT_ID=order-service
OUTBOX_KAFKA_CLIENT_ID=outbox-worker
INVENTORY_KAFKA_GROUP_ID=inventory-service
```

---

## 十四、配置校验

配置加载后要校验必填项：

```go
func (c Config) ValidateForInventoryService() error {
    if c.DatabaseURL == "" {
        return errors.New("DATABASE_URL is required")
    }
    if len(c.KafkaBrokers) == 0 || c.KafkaBrokers[0] == "" {
        return errors.New("KAFKA_BROKERS is required")
    }
    if c.KafkaGroupID == "" {
        return errors.New("KAFKA_GROUP_ID is required")
    }
    return nil
}
```

不要等服务运行到一半才发现配置为空。

---

## 十五、优雅退出模板

Kafka consumer 和 outbox worker 都需要响应退出信号：

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

if err := worker.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
    log.Fatal(err)
}
```

HTTP 服务可以这样关闭：

```go
srv := &http.Server{Addr: cfg.HTTPAddr, Handler: mux}

go func() {
    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    _ = srv.Shutdown(shutdownCtx)
}()
```

优雅退出的目标是：

```text
停止接收新请求。
让正在处理的请求或消息尽量完成。
关闭数据库和 Kafka 客户端。
```

---

## 十六、包依赖方向

建议依赖方向：

```text
cmd -> internal/order
cmd -> internal/inventory
cmd -> internal/outbox
业务包 -> internal/kafka 抽象接口
业务包 -> repository
repository -> pgx
```

不要让 `internal/order` 直接依赖 `internal/inventory`。订单服务发布事件，库存服务消费事件，二者通过 Kafka 事件协作。

---

## 十七、常见错误

### 1. main.go 越写越大

`main.go` 只负责组装和启动，不写业务逻辑。业务逻辑放到 service 和 handler。

### 2. 配置散落在各处

不要在 repository、producer、consumer 里各自读取环境变量。统一在启动时加载配置，再显式传入。

### 3. 没有 context

数据库查询、Kafka 发送、HTTP 请求都应该传递 `context.Context`，否则超时和退出不好控制。

### 4. 业务代码直接依赖具体 Kafka 客户端

可以先简单依赖，但更好的做法是定义小接口，方便测试：

```go
type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
}
```
