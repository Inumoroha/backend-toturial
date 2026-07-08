# 3. 测试策略、Swagger 与 Makefile 的整体安排

本节目标：先建立测试、接口文档和工程命令的整体认识，再进入后续具体实践。

很多人学习测试时会直接问：“这个 handler 的测试怎么写？”这当然要学，但更重要的是先搞清楚：后端项目到底有哪些层需要测试，每一层测什么，不测什么。

---

## 一、Gin 项目常见测试层次

一个分层后的 Gin 项目通常有这些层：

```text
HTTP 请求
  ↓
router / middleware
  ↓
handler
  ↓
service
  ↓
repository
  ↓
database / redis / external api
```

对应测试可以分为：

```text
handler 测试：关注 HTTP 行为
service 测试：关注业务规则
repository 测试：关注数据库读写
集成测试：关注多个组件组合后是否正常
```

本阶段重点是 handler 测试和 service 测试。repository 测试会涉及测试数据库、事务回滚、数据隔离，适合在项目实战阶段继续深入。

---

## 二、handler 测试测什么

handler 位于 HTTP 入口处，它不应该承载太多业务规则。handler 测试重点是：

- 路由是否注册正确。
- HTTP 方法是否正确。
- 请求参数绑定是否正确。
- 参数错误时是否返回统一错误响应。
- 未登录时是否被中间件拦截。
- service 返回错误时，handler 是否转换为正确状态码。
- 响应 JSON 结构是否符合约定。

handler 测试不应该深度测试数据库。否则每个 handler 测试都会变慢、变脆。

---

## 三、service 测试测什么

service 层承载业务规则。例如：

- 注册用户时邮箱不能重复。
- 创建任务时标题不能为空。
- 普通用户不能删除别人的资源。
- 更新资料时只能修改允许的字段。

service 测试重点是：

- 输入不同参数时，业务结果是否正确。
- repository 返回错误时，service 是否正确处理。
- 边界条件是否覆盖。
- 错误类型是否符合预期。

service 测试通常使用 mock 或 fake repository，不直接依赖真实数据库。

---

## 四、Swagger 解决什么问题

Swagger/OpenAPI 用来描述 HTTP API。它能告诉前端、测试和其他后端同事：

- 接口路径是什么。
- 请求方法是什么。
- 请求参数有哪些。
- 请求体结构是什么。
- 成功响应长什么样。
- 错误响应长什么样。
- 是否需要认证。

Swagger 不是替代测试的工具。它解决“接口如何被理解和调用”的问题；测试解决“接口行为是否正确”的问题。

---

## 五、Makefile 解决什么问题

当项目命令越来越多时，如果每次都手敲完整命令，很容易出错。

例如：

```bash
go run ./cmd/server
go test ./...
go test ./... -cover
swag init -g cmd/server/main.go -o docs
golangci-lint run
```

可以用 Makefile 统一成：

```bash
make run
make test
make test-cover
make swagger
make lint
```

这样新同事不需要记住所有细节，只需要看 README 中的命令。

---

## 六、推荐 Makefile

项目根目录创建 `Makefile`：

```makefile
.PHONY: run test test-cover swagger tidy lint

run:
	go run ./cmd/server

test:
	go test ./...

test-cover:
	go test ./... -cover

swagger:
	swag init -g cmd/server/main.go -o docs

tidy:
	go mod tidy

lint:
	golangci-lint run
```

注意：Makefile 中命令前面必须是 Tab，不是空格。

Windows 用户如果没有安装 `make`，可以先直接执行对应命令。后续也可以用 PowerShell 脚本替代，但学习阶段建议理解 Makefile 的思想。

---

## 七、推荐测试目录规则

Go 的测试文件一般和被测试代码放在同一目录：

```text
internal/
├── handler/
│   ├── user_handler.go
│   └── user_handler_test.go
├── service/
│   ├── user_service.go
│   └── user_service_test.go
└── repository/
    ├── user_repository.go
    └── user_repository_test.go
```

测试文件命名规则：

```text
xxx_test.go
```

测试函数命名规则：

```go
func TestUserService_Register(t *testing.T) {}
func TestUserHandler_CreateUser(t *testing.T) {}
```

名称建议包含“被测对象 + 被测行为”，不要只写 `TestUser`。

---

## 八、测试数据如何管理

学习阶段建议：

- handler 测试中用 fake service。
- service 测试中用 fake repository。
- 不要一开始就引入复杂 mock 框架。
- 表格驱动测试优先。
- 每个测试只验证一个明确行为。

例如：

```go
tests := []struct {
    name    string
    input   RegisterInput
    wantErr bool
}{
    {
        name: "valid input",
        input: RegisterInput{
            Email: "a@example.com",
            Password: "123456",
        },
        wantErr: false,
    },
    {
        name: "empty email",
        input: RegisterInput{
            Email: "",
            Password: "123456",
        },
        wantErr: true,
    },
}
```

表格驱动测试的好处是：新增场景时，只需要加一行测试用例。

---

## 九、阶段推进顺序

接下来几节会按这个顺序推进：

1. 先写 handler 测试，掌握 `httptest`。
2. 再写 service 表格驱动测试，掌握业务规则验证。
3. 然后补 Swagger 注释，生成接口文档。
4. 最后用 Makefile 和 README 把工程命令收口。

这个顺序比较自然，因为测试能反过来暴露接口设计问题，Swagger 则需要在接口稳定后再补充。

---

## 十、常见误区

### 1. 有 Swagger 就不用测试

不对。Swagger 只能说明接口是什么，不能证明接口真的正确。

### 2. 所有测试都连真实数据库

不建议。这样测试慢，而且需要准备复杂环境。业务规则可以先用 fake repository 测。

### 3. 测试越多越好

测试要覆盖关键行为，而不是机械追求数量。低价值测试会增加维护成本。

### 4. Makefile 只是 Linux 用户才需要

不完全是。Makefile 的核心价值是统一命令。Windows 上即使暂时不用 make，也应该把标准命令写清楚。

---

## 十一、练习

请先不用写代码，完成下面的设计练习：

1. 列出你的用户模块中哪些行为适合 handler 测试。
2. 列出哪些业务规则适合 service 测试。
3. 写出你希望 README 中包含的 5 个常用命令。
4. 为注册接口设计 Swagger 需要描述的请求字段和响应字段。

---

## 十二、验收标准

本节完成后，请确认你能回答：

- handler 测试和 service 测试的区别是什么。
- Swagger 和测试分别解决什么问题。
- Makefile 在项目中负责什么。
- 为什么测试不应该全部依赖真实数据库。

理解这些之后，再进入下一节 handler 测试实践。
