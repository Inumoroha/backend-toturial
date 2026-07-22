# 6. Swagger 注释与接口文档

本节目标：为 Gin 项目添加 Swagger/OpenAPI 文档，让接口可以被浏览器查看、调试和协作沟通。

当项目只有两三个接口时，口头说明还能勉强支撑。一旦接口变多，就必须有文档。Swagger 的价值不是“好看”，而是让接口契约变得清晰。

---

## 一、安装 swag 工具

Gin 项目常用 `swaggo` 生成 Swagger 文档。

安装命令：

```bash
go install github.com/swaggo/swag/cmd/swag@latest
```

安装 Gin Swagger 依赖：

```bash
go get github.com/swaggo/gin-swagger
go get github.com/swaggo/files
```

验证 `swag` 是否可用：

```bash
swag --version
```

如果提示找不到命令，检查 Go bin 目录是否加入 PATH：

```bash
go env GOPATH
```

通常可执行文件位于：

```text
$GOPATH/bin
```

---

## 二、在 main 中添加项目信息

在 `cmd/server/main.go` 顶部添加 Swagger 注释：

```go
// @title Gin User API
// @version 1.0
// @description Gin 后端学习项目接口文档
// @host localhost:8080
// @BasePath /api/v1
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
func main() {
    // ...
}
```

这些注释会成为 Swagger 文档的全局信息。

字段说明：

- `@title`：接口文档标题。
- `@version`：接口版本。
- `@description`：项目说明。
- `@host`：默认访问地址。
- `@BasePath`：接口基础路径。
- `@securityDefinitions.apikey`：定义认证方式。
- `@name Authorization`：token 放在请求头 Authorization 中。

---

## 三、注册 Swagger 路由

在路由初始化处添加：

```go
import (
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"

    _ "gin-user-api/docs"
)
```

注册路由：

```go
r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
```

注意 `_ "gin-user-api/docs"` 是匿名导入，用来加载生成的 Swagger 文档。

---

## 四、为请求和响应结构体写注释

示例请求：

```go
type RegisterRequest struct {
    Email    string `json:"email" binding:"required,email" example:"a@example.com"`
    Password string `json:"password" binding:"required,min=6" example:"123456"`
    Nickname string `json:"nickname" binding:"omitempty,max=20" example:"alice"`
}
```

示例响应：

```go
type UserResponse struct {
    ID    uint   `json:"id" example:"1"`
    Email string `json:"email" example:"a@example.com"`
}
```

统一响应也可以定义：

```go
type APIResponse struct {
    Code    string      `json:"code" example:"OK"`
    Message string      `json:"message" example:"success"`
    Data    interface{} `json:"data,omitempty"`
}
```

`example` 不是必须，但很建议写。它会让 Swagger 页面更容易理解。

---

## 五、为注册接口添加 Swagger 注释

```go
// Register 用户注册
// @Summary 用户注册
// @Description 使用邮箱、密码和昵称创建新用户
// @Tags 用户
// @Accept json
// @Produce json
// @Param request body RegisterRequest true "注册请求"
// @Success 200 {object} response.Response{data=UserResponse}
// @Failure 400 {object} response.Response
// @Failure 409 {object} response.Response
// @Router /users/register [post]
func (h *UserHandler) Register(c *gin.Context) {
    // ...
}
```

常用注释解释：

- `@Summary`：一句话说明接口。
- `@Description`：更详细说明。
- `@Tags`：接口分组。
- `@Accept`：请求内容类型。
- `@Produce`：响应内容类型。
- `@Param`：参数说明。
- `@Success`：成功响应。
- `@Failure`：失败响应。
- `@Router`：路径和方法。

---

## 六、为需要认证的接口添加注释

示例：

```go
// Profile 获取当前用户资料
// @Summary 获取当前用户资料
// @Tags 用户
// @Produce json
// @Security BearerAuth
// @Success 200 {object} response.Response{data=UserResponse}
// @Failure 401 {object} response.Response
// @Router /profile [get]
func (h *UserHandler) Profile(c *gin.Context) {
    // ...
}
```

`@Security BearerAuth` 表示这个接口需要认证。

在 Swagger 页面测试时，Authorization 通常填：

```text
Bearer your-jwt-token
```

---

## 七、生成文档

执行：

```bash
swag init -g cmd/server/main.go -o docs
```

生成后会出现：

```text
docs/
├── docs.go
├── swagger.json
└── swagger.yaml
```

启动服务：

```bash
go run ./cmd/server
```

浏览器访问：

```text
http://localhost:8080/swagger/index.html
```

如果能看到 Swagger 页面，说明文档路由生效。

---

## 八、常见问题

### 1. `swag` 命令找不到

确认 Go bin 目录在 PATH 中。PowerShell 可以查看：

```powershell
go env GOPATH
```

然后把 `GOPATH\bin` 加入系统 PATH。

### 2. 生成时报 cannot find type

检查 Swagger 注释中的类型名是否写对，尤其是跨包类型。必要时使用完整包路径或在当前包中定义文档专用响应结构。

### 3. 页面打开但接口为空

检查 handler 上方是否真的有 Swagger 注释，并且执行过：

```bash
swag init -g cmd/server/main.go -o docs
```

### 4. BasePath 和真实路由不一致

如果你真实路由是：

```go
api := r.Group("/api/v1")
```

那么 main 注释应写：

```go
// @BasePath /api/v1
```

handler 的 `@Router` 只写 BasePath 后面的部分。

---

## 九、练习

请为以下接口补充 Swagger 注释：

1. 用户注册。
2. 用户登录。
3. 获取当前用户资料。
4. 修改当前用户资料。
5. 用户列表分页查询。

每个接口至少写：

- `@Summary`
- `@Tags`
- `@Param`
- `@Success`
- `@Failure`
- `@Router`

需要登录的接口加：

```go
// @Security BearerAuth
```

---

## 十、验收标准

本节完成后，请确认：

- `swag --version` 可用。
- 项目存在 `docs/swagger.json`。
- `/swagger/index.html` 可以访问。
- 至少 5 个核心接口出现在 Swagger 页面。
- 需要登录的接口显示认证要求。
- Swagger 中的路径和真实路由一致。

Swagger 完成后，最后用综合实践把配置、日志、测试、文档和 Makefile 收口。

