# 1. Viper 配置与环境区分

本节目标：把项目中的端口、数据库连接、JWT 密钥、日志级别等信息从代码中移出来，交给配置文件和环境变量管理。

在真实项目里，配置管理是后端工程的基本功。一个合格的 Gin 项目不应该把这些内容写死在代码里：

```go
r.Run(":8080")
dsn := "root:123456@tcp(127.0.0.1:3306)/app"
secret := "dev-secret"
```

这样写在学习阶段没问题，但项目一旦要多人协作、测试、部署，就会马上出现问题。

---

## 一、本节要解决的问题

本节完成后，你的项目应该支持：

- 本地开发读取 `config/config.local.yaml`。
- 测试环境读取 `config/config.test.yaml`。
- 默认配置读取 `config/config.yaml`。
- 生产环境可以用环境变量覆盖配置。
- Go 代码中通过结构体访问配置，而不是到处读取字符串。

---

## 二、安装 Viper

在项目根目录执行：

```bash
go get github.com/spf13/viper
```

如果你还没有初始化 Go module，先执行：

```bash
go mod init gin-user-api
```

验证依赖是否安装成功：

```bash
go list -m github.com/spf13/viper
```

能看到版本号即可。

---

## 三、编写配置文件

创建目录：

```bash
mkdir -p config
```

Windows PowerShell 可以使用：

```powershell
New-Item -ItemType Directory -Force config
```

创建 `config/config.yaml`：

```yaml
app:
  name: "gin-user-api"
  env: "local"

server:
  port: 8080
  mode: "debug"

database:
  dsn: "root:password@tcp(127.0.0.1:3306)/gin_user_api?charset=utf8mb4&parseTime=True&loc=Local"
  max_idle_conns: 10
  max_open_conns: 100

jwt:
  secret: "change-me-in-production"
  expire_hours: 24

log:
  level: "debug"
  format: "console"
```

再创建 `config/config.test.yaml`：

```yaml
app:
  name: "gin-user-api"
  env: "test"

server:
  port: 18080
  mode: "test"

database:
  dsn: "root:password@tcp(127.0.0.1:3306)/gin_user_api_test?charset=utf8mb4&parseTime=True&loc=Local"
  max_idle_conns: 2
  max_open_conns: 5

jwt:
  secret: "test-secret"
  expire_hours: 1

log:
  level: "warn"
  format: "json"
```

配置文件的核心原则是：同一类配置保持相同结构，不同环境只改变值。

---

## 四、定义配置结构体

创建 `internal/config/config.go`：

```go
package config

type Config struct {
    App      AppConfig      `mapstructure:"app"`
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    JWT      JWTConfig      `mapstructure:"jwt"`
    Log      LogConfig      `mapstructure:"log"`
}

type AppConfig struct {
    Name string `mapstructure:"name"`
    Env  string `mapstructure:"env"`
}

type ServerConfig struct {
    Port int    `mapstructure:"port"`
    Mode string `mapstructure:"mode"`
}

type DatabaseConfig struct {
    DSN          string `mapstructure:"dsn"`
    MaxIdleConns int    `mapstructure:"max_idle_conns"`
    MaxOpenConns int    `mapstructure:"max_open_conns"`
}

type JWTConfig struct {
    Secret      string `mapstructure:"secret"`
    ExpireHours int   `mapstructure:"expire_hours"`
}

type LogConfig struct {
    Level  string `mapstructure:"level"`
    Format string `mapstructure:"format"`
}
```

这里的 `mapstructure` tag 用来告诉 Viper：YAML 中的字段应该映射到哪个 Go 字段。

---

## 五、编写加载配置函数

继续在 `internal/config/config.go` 中添加：

```go
package config

import (
    "fmt"
    "strings"

    "github.com/spf13/viper"
)

func Load(path string) (*Config, error) {
    v := viper.New()

    v.SetConfigFile(path)
    v.SetConfigType("yaml")

    v.SetEnvPrefix("APP")
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    v.AutomaticEnv()

    setDefaults(v)

    if err := v.ReadInConfig(); err != nil {
        return nil, fmt.Errorf("read config: %w", err)
    }

    var cfg Config
    if err := v.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshal config: %w", err)
    }

    return &cfg, nil
}

func setDefaults(v *viper.Viper) {
    v.SetDefault("server.port", 8080)
    v.SetDefault("server.mode", "debug")
    v.SetDefault("log.level", "info")
    v.SetDefault("log.format", "json")
    v.SetDefault("database.max_idle_conns", 10)
    v.SetDefault("database.max_open_conns", 100)
}
```

关键点解释：

- `viper.New()`：创建独立实例，避免全局配置污染测试。
- `SetConfigFile(path)`：明确指定配置文件路径。
- `SetEnvPrefix("APP")`：环境变量统一以 `APP_` 开头。
- `SetEnvKeyReplacer`：把 `database.dsn` 转成 `APP_DATABASE_DSN`。
- `AutomaticEnv()`：允许环境变量覆盖配置。
- `SetDefault()`：提供默认值，避免配置缺失导致程序直接崩溃。

---

## 六、在 main 中使用配置

示例 `cmd/server/main.go`：

```go
package main

import (
    "fmt"
    "log"

    "github.com/gin-gonic/gin"

    "gin-user-api/internal/config"
)

func main() {
    cfg, err := config.Load("config/config.yaml")
    if err != nil {
        log.Fatalf("load config failed: %v", err)
    }

    gin.SetMode(cfg.Server.Mode)

    r := gin.New()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
            "env":     cfg.App.Env,
        })
    })

    addr := fmt.Sprintf(":%d", cfg.Server.Port)
    if err := r.Run(addr); err != nil {
        log.Fatalf("start server failed: %v", err)
    }
}
```

现在端口不再写死在代码里，而是来自配置文件。

---

## 七、用环境变量覆盖配置

假设配置文件中端口是 `8080`，你可以临时覆盖：

Linux/macOS：

```bash
APP_SERVER_PORT=9090 go run ./cmd/server
```

PowerShell：

```powershell
$env:APP_SERVER_PORT="9090"
go run ./cmd/server
```

访问：

```bash
curl http://localhost:9090/ping
```

如果能返回 `pong`，说明环境变量覆盖成功。

---

## 八、本地、测试、生产如何区分

推荐规则：

```text
本地开发：config/config.local.yaml
自动化测试：config/config.test.yaml
默认示例：config/config.yaml
生产环境：配置文件 + 环境变量覆盖敏感项
```

生产环境尤其要注意：

- 不要把真实数据库密码提交到 Git。
- 不要把生产 JWT Secret 写进仓库。
- 不要把云服务密钥写进 YAML。
- 敏感配置优先使用环境变量或密钥管理服务。

---

## 九、常见问题

### 1. 修改 YAML 后不生效

检查是否加载的是正确路径：

```go
config.Load("config/config.yaml")
```

如果你实际修改的是 `config.local.yaml`，但代码读取的是 `config.yaml`，自然不会生效。

### 2. 环境变量覆盖失败

检查命名是否符合规则：

```text
server.port       -> APP_SERVER_PORT
database.dsn      -> APP_DATABASE_DSN
jwt.secret        -> APP_JWT_SECRET
```

还要确认 PowerShell 中设置环境变量后，是在同一个终端里启动程序。

### 3. YAML 字段读不到结构体里

检查 `mapstructure` tag 是否一致。例如：

```go
MaxOpenConns int `mapstructure:"max_open_conns"`
```

对应 YAML：

```yaml
max_open_conns: 100
```

---

## 十、练习

请完成以下练习：

1. 给配置增加 `server.read_timeout_seconds`。
2. 给配置增加 `server.write_timeout_seconds`。
3. 在 main 中打印当前运行环境 `cfg.App.Env`。
4. 使用环境变量把 `jwt.secret` 覆盖为 `local-secret-123`。
5. 创建 `config/config.local.yaml`，让本地端口使用 `8081`。

---

## 十一、验收标准

本节完成后，请确认：

- 项目中已经安装 Viper。
- 配置文件能被读取到结构体。
- 修改 YAML 中的端口后，服务启动端口改变。
- 环境变量 `APP_SERVER_PORT` 可以覆盖 YAML。
- 敏感配置没有继续硬编码在 handler、service 或 main 中。

达到这些要求后，再进入下一节 zap 日志。
