# 07-02 测试策略、Makefile 与目录规范

本节目标：把测试和工程命令讲得更可执行。你不需要一开始就写复杂测试，但必须知道哪些地方最值得测。

## 一、先测什么

测试优先级：

1. service 中的业务规则。
2. handler 的状态码和响应格式。
3. repository 的数据库查询。
4. middleware 的认证行为。

不要一开始就追求 100% 覆盖率。先覆盖最容易改坏、最影响业务的地方。

## 二、service 测试结构

推荐文件：

```text
internal/service/task_service_test.go
```

测试命名：

```go
func TestTaskService_Create_DefaultStatus(t *testing.T) {}
func TestTaskService_Create_InvalidStatus(t *testing.T) {}
func TestTaskService_List_DefaultPagination(t *testing.T) {}
```

一个测试只验证一个重点。测试名要说明场景和预期。

## 三、表格驱动测试

Go 常用表格驱动测试：

```go
func TestIsValidTaskStatus(t *testing.T) {
    tests := []struct {
        name string
        status string
        want bool
    }{
        {name: "todo", status: "todo", want: true},
        {name: "doing", status: "doing", want: true},
        {name: "done", status: "done", want: true},
        {name: "invalid", status: "unknown", want: false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := isValidTaskStatus(tt.status)
            if got != tt.want {
                t.Fatalf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

适合测试多个输入输出组合。

## 四、handler 测试关注点

handler 测试不应该关心数据库细节，重点是：

- 参数错误返回 400。
- 未登录返回 401。
- 无权限返回 403。
- 资源不存在返回 404。
- 成功返回统一响应。

示例断言响应体：

```go
var body response.Body
if err := json.Unmarshal(w.Body.Bytes(), &body); err != nil {
    t.Fatal(err)
}
if body.Code != 40001 {
    t.Fatalf("expected code 40001, got %d", body.Code)
}
```

## 五、测试数据库

repository 测试可以使用独立测试数据库。

建议：

- 不要用生产库。
- 每次测试前清理表。
- 测试数据自己创建。
- 测试结束后清理。

学习阶段如果觉得数据库测试太重，可以先把重点放在 service 和 handler。

## 六、Makefile 完整示例

```makefile
.PHONY: run test fmt vet tidy

run:
	go run ./cmd/server

test:
	go test ./...

fmt:
	gofmt -w .

vet:
	go vet ./...

tidy:
	go mod tidy

check: fmt vet test
```

常用：

```bash
make check
```

这条命令适合在提交代码前运行。

## 七、README 应该写什么

项目 README 至少包含：

- 项目简介。
- 技术栈。
- 目录结构。
- 配置说明。
- 本地启动。
- Docker 启动。
- 测试命令。
- 接口文档地址。
- 常见问题。

别人能不能顺利启动你的项目，README 非常关键。

## 八、本节练习

完成：

- 为任务状态校验写表格驱动测试。
- 为注册参数错误写 handler 测试。
- 为认证中间件未带 token 写测试。
- 编写 Makefile。
- 在 README 写清楚本地启动步骤。

## 九、验收清单

你应该能够回答：

- 为什么先测试 service？
- 什么是表格驱动测试？
- handler 测试主要验证什么？
- repository 测试为什么要使用测试库？
- `make check` 适合什么时候运行？

