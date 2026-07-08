# 07-01 配置、日志、测试与 Swagger 文档

本阶段目标：让项目从练习代码走向工程代码。一个真实后端服务必须能配置、能排错、能测试、能生成接口文档。

## 一、配置管理

不要把端口、数据库密码、JWT secret 写死在代码里。

推荐先使用 YAML 配置：

```text
configs/config.yaml
```

示例：

```yaml
server:
  port: 8080

database:
  dsn: "root:password@tcp(127.0.0.1:3306)/gin_learning?charset=utf8mb4&parseTime=True&loc=Local"

jwt:
  secret: "change-me-in-production"
  expire_hours: 24

log:
  level: "info"
```

安装配置库：

```bash
go get github.com/spf13/viper
```

加载配置：

```go
type Config struct {
    Server struct {
        Port int `mapstructure:"port"`
    } `mapstructure:"server"`
    Database struct {
        DSN string `mapstructure:"dsn"`
    } `mapstructure:"database"`
    JWT struct {
        Secret      string `mapstructure:"secret"`
        ExpireHours int    `mapstructure:"expire_hours"`
    } `mapstructure:"jwt"`
}

func Load(path string) (*Config, error) {
    viper.SetConfigFile(path)
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }

    var cfg Config
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

生产环境可以使用环境变量覆盖敏感配置，例如数据库密码和 JWT secret。

## 二、日志

不要只用 `fmt.Println`。真实项目需要结构化日志。

推荐使用 `zap`：

```bash
go get go.uber.org/zap
```

初始化：

```go
logger, err := zap.NewProduction()
if err != nil {
    panic(err)
}
defer logger.Sync()

logger.Info("server starting", zap.Int("port", 8080))
```

日志应该包含：

- 请求路径。
- 请求方法。
- 状态码。
- 请求耗时。
- 错误信息。
- request id。

## 三、请求日志中间件

```go
func AccessLog(logger *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()

        logger.Info("http request",
            zap.String("method", c.Request.Method),
            zap.String("path", c.Request.URL.Path),
            zap.Int("status", c.Writer.Status()),
            zap.Duration("latency", time.Since(start)),
        )
    }
}
```

注册：

```go
r.Use(middleware.AccessLog(logger))
```

## 四、测试 handler

Go 标准库提供了 `httptest`。

示例：

```go
func TestPing(t *testing.T) {
    gin.SetMode(gin.TestMode)

    r := gin.New()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "pong"})
    })

    req := httptest.NewRequest(http.MethodGet, "/ping", nil)
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("expected status 200, got %d", w.Code)
    }
}
```

测试的价值不是为了追求覆盖率数字，而是保护核心行为不被改坏。

## 五、测试 service

service 测试通常不需要真正启动 HTTP 服务。

如果 service 依赖 repository，可以定义接口：

```go
type UserRepository interface {
    GetByID(id uint) (*model.User, error)
    Create(user *model.User) error
}
```

这样测试时可以传入 fake repository。

好处：

- 测试更快。
- 不依赖数据库。
- 更容易覆盖异常分支。

## 六、Swagger 文档

接口文档能减少前后端沟通成本。

常用工具：

```bash
go install github.com/swaggo/swag/cmd/swag@latest
go get github.com/swaggo/gin-swagger
go get github.com/swaggo/files
```

在 handler 上写注释：

```go
// ListUsers godoc
// @Summary 查询用户列表
// @Tags users
// @Produce json
// @Success 200 {object} response.Body
// @Router /api/v1/users [get]
func (h *UserHandler) List(c *gin.Context) {
    response.OK(c, nil)
}
```

生成：

```bash
swag init -g cmd/server/main.go
```

注册文档路由：

```go
r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
```

访问：

```text
http://localhost:8080/swagger/index.html
```

## 七、Makefile

常用命令整理到 Makefile：

```makefile
.PHONY: run test fmt

run:
	go run ./cmd/server

test:
	go test ./...

fmt:
	gofmt -w .
```

后续可以加入：

```makefile
lint:
	golangci-lint run
```

## 八、阶段练习

完成：

- 使用 YAML 管理配置。
- 数据库 DSN 从配置读取。
- JWT secret 从配置读取。
- 使用 zap 打印启动日志和请求日志。
- 为至少 3 个 handler 写测试。
- 为至少 3 个 service 方法写测试。
- 生成 Swagger 文档。
- 编写 Makefile。

## 九、验收清单

你应该能够回答：

- 为什么配置不能写死在代码里？
- 结构化日志比普通字符串日志好在哪里？
- handler 测试和 service 测试有什么区别？
- 什么情况下需要 mock 或 fake？
- Swagger 文档如何生成？

