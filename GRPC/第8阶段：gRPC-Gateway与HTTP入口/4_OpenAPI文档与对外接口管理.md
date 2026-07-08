# 4. OpenAPI 文档与对外接口管理

本节目标：了解如何从 proto 生成 OpenAPI 文档，并管理对外 API 的兼容性。

当 gRPC-Gateway 对外提供 HTTP API 时，调用方需要文档。OpenAPI 可以描述路径、参数、请求体、响应和错误，是对外协作的重要材料。

---

## 一、为什么需要 OpenAPI

- 前端可以据此了解接口字段。
- 第三方调用方可以据此接入。
- 测试工具可以导入 OpenAPI。
- 团队可以评审接口变更。

proto 是源头，OpenAPI 可以由 proto 生成，避免手写文档和实现脱节。

---

## 二、安装插件

```bash
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```

生成：

```bash
mkdir -p openapi
protoc -I . --openapiv2_out ./openapi proto/user/v1/user.proto
```

---

## 三、文档应该补充什么

生成的 OpenAPI 只是基础，还需要补充：

- 认证方式。
- 错误响应格式。
- 分页规则。
- 字段含义。
- 版本策略。
- 限流规则。

这些内容可以通过 proto 注释、额外文档或 API 网关文档平台维护。

---

## 四、常见问题

- 生成文档后不检查：注释和 schema 可能不符合对外表达。
- 文档和实现脱节：最好从 proto 统一生成。
- 不记录版本策略：调用方不知道何时升级。
- 对外 API 频繁 breaking change：第三方接入会很痛苦。

---

## 五、练习任务

1. 为 UserService 生成 OpenAPI。
2. 给 proto message 和 rpc 增加注释。
3. 写一页 API 调用说明。
4. 标注错误码和认证方式。

---

## 六、完成标准

- 能生成基本 OpenAPI 文档。
- 文档能辅助 HTTP 调用方接入。
- 知道对外 API 兼容性更敏感。


---

## 七、完整操作步骤

本节目标：从 proto 生成 OpenAPI 文档，并理解对外 HTTP API 的管理规则。

操作步骤：

1. 安装 `protoc-gen-openapiv2`。
2. 创建 `openapi` 输出目录。
3. 使用 protoc 生成 swagger json。
4. 检查生成文件。
5. 在 proto 中补充注释。
6. 重新生成文档。
7. 为认证、错误格式、分页规则写补充说明。
8. 把 OpenAPI 文档作为对外接口交付物。

---

## 八、安装 OpenAPI 插件

```powershell
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```

验证：

```powershell
Get-Command protoc-gen-openapiv2
```

如果找不到：

```powershell
$env:Path += ";$(go env GOPATH)\bin"
```

---

## 九、生成 OpenAPI 文档

创建输出目录：

```powershell
New-Item -ItemType Directory -Force openapi
```

生成：

```powershell
protoc -I . -I third_party `
  --openapiv2_out=openapi `
  proto/user/v1/user.proto
```

检查：

```powershell
Get-ChildItem openapi
```

预期看到类似：

```text
user.swagger.json
```

不同插件版本可能输出文件名略有差异，重点是 openapi 目录下出现 json 文档。

---

## 十、查看文档内容

PowerShell：

```powershell
Get-Content openapi/user.swagger.json -Head 40
```

你应该能看到：

```json
{
  "swagger": "2.0",
  "info": {
    "title": "user.proto",
    "version": "version not set"
  },
  "paths": {
    "/v1/users/{id}": {
      "get": {
```

这说明 HTTP annotation 已经被转换成 OpenAPI path。

---

## 十一、给 proto 补充注释

```proto
// UserService provides user query and creation APIs.
service UserService {
  // GetUser returns a user by id.
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"
    };
  }

  // CreateUser creates a new user.
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {
    option (google.api.http) = {
      post: "/v1/users"
      body: "*"
    };
  }
}
```

注释能帮助文档使用者理解接口含义。不要只生成字段结构，却没有业务说明。

---

## 十二、对外接口文档还要补什么

OpenAPI 生成的是基础结构，但真实对外文档还需要说明：

```text
认证方式：Authorization: Bearer <token>
trace id：X-Trace-Id
错误响应格式：code/message/traceId
分页规则：page/pageSize 或 cursor
限流规则：HTTP 429
版本策略：/v1、/v2 如何演进
字段兼容：新增字段、废弃字段如何处理
```

这些内容可以写在 README、接口文档平台，或通过 OpenAPI extension 补充。

---

## 十三、用文档检查路由

生成 OpenAPI 后，重点检查：

1. 路径是否正确。
2. HTTP 方法是否正确。
3. path 参数是否正确。
4. request body 是否正确。
5. response schema 是否正确。
6. 错误响应是否需要补充。

例如如果你期望：

```text
GET /v1/users/{id}
```

但文档里没有这个 path，说明 annotation 或生成命令有问题。

---

## 十四、常见错误排查

### 1. openapi 目录没有文件

检查是否安装插件，以及命令中是否有：

```text
--openapiv2_out=openapi
```

### 2. 文档里没有路径

检查 proto 中是否添加了 `google.api.http` annotation。

### 3. 文档字段说明太少

补充 proto 注释，或维护额外 API 说明。

### 4. 文档和实现不一致

不要手写一份完全独立的 API 文档却长期不更新。尽量从 proto 生成，再补充说明。

---

## 十五、API 兼容性管理

对外 HTTP API 比内部 gRPC 更敏感。建议规则：

```text
可以新增响应字段
不要随意删除字段
不要随意改字段类型
不要随意改路径含义
废弃字段先标注 deprecated，再给迁移期
重大变化开 /v2
```

proto 中可以标记字段废弃：

```proto
string old_email = 4 [deprecated = true];
```

HTTP 路由重大变化时，建议新开版本：

```text
/v1/users/{id}
/v2/users/{id}
```

---

## 十六、练习任务

1. 安装 `protoc-gen-openapiv2`。
2. 生成 `openapi/user.swagger.json`。
3. 检查文档中是否包含 `/v1/users/{id}`。
4. 给 `GetUser` 和 `CreateUser` 添加注释。
5. 写一段认证和错误响应说明。
6. 设计 v1 到 v2 的升级策略。

---

## 十七、完成标准

完成本节后，你应该能：

```text
从 proto 生成 OpenAPI 文档
检查 OpenAPI 中的 paths 和 schema
通过 proto 注释改善文档可读性
说明认证、错误、分页、版本策略
理解对外 API 的兼容性约束
知道什么时候需要 /v2
```

---

## 教程闭环检查

1. **完整操作步骤**：安装插件、生成 OpenAPI、检查 paths、补充注释、记录接口规则。
2. **完整代码**：使用正文中的注释和 OpenAPI 生成命令。
3. **运行命令**：执行 protoc --openapiv2_out，并用 Get-Content 查看文档。
4. **预期输出**：openapi 目录出现 swagger json，文档包含 `/v1/users/{id}`。
5. **常见错误排查**：重点检查插件、annotation、输出目录、文档与实现不一致。
6. **练习任务**：完成注释、认证说明、错误响应说明和版本策略。
7. **完成标准**：能把 gRPC-Gateway 暴露的 HTTP API 变成可交付文档。