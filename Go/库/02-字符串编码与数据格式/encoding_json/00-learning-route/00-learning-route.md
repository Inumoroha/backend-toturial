# 00. 学习路线

本节目标：学完后你能知道学习 `encoding/json` 的顺序，并能把每个知识点对应到真实后端项目里的使用场景。

简短引入：后端工程师使用 JSON，最常见的地方是 HTTP 请求体、HTTP 响应、配置文件、日志、消息队列事件和数据库中的 JSON 字段。学习路线应该围绕这些场景展开，而不是从所有 API 名称开始背。

## 一、为什么需要它

在真实项目里，JSON 是服务和外界交换数据的边界。比如前端提交创建订单请求，后端要把 JSON 请求体解析成 Go 结构体；后端返回用户信息时，又要把结构体转成 JSON。

可以把 `encoding/json` 理解为三层能力：

1. 把 Go 值转换成 JSON。
2. 把 JSON 转换成 Go 值。
3. 在接口边界上控制字段、错误、精度和兼容性。

第一层很快能学会，真正决定项目质量的是第三层。

```text
JSON 不是简单的字符串拼接，它是接口契约的一部分。
```

## 二、基本用法

这份路线的用法很简单：先按编号阅读，不跳过基础篇；每读完一篇就运行里面的最小示例；最后用订单小项目把知识串起来。

不要把它当成一次性读完的资料，更适合当成一套训练计划。真实学习时，你可以每次只推进一个主题，比如今天只练“请求体解析”，明天再练“错误处理”。

## 三、关键参数/语法/代码结构

这套教程围绕 `encoding/json` 在后端项目中的常用能力展开，而不是按标准库 API 顺序机械罗列。

核心学习结构是：

```text
结构体和标签
HTTP 请求与响应
错误处理与严格解析
流式数据与动态 JSON
数字精度与自定义编解码
接口契约与订单项目
```

下面是具体阶段安排。

### 第一阶段：先会读写 JSON

对应文件：

- `01-basics/01-marshal-unmarshal.md`
- `01-basics/02-struct-tags.md`

你需要掌握：

- `json.Marshal` 把结构体转成 JSON。
- `json.Unmarshal` 把 JSON 转成结构体。
- `json` 标签控制字段名、忽略字段和 `omitempty`。

这一阶段先不追求复杂封装，目标是能写出清晰可运行的基础代码。

### 第二阶段：放进 HTTP 接口

对应文件：

- `02-api-input-output/03-http-response-json.md`
- `02-api-input-output/04-request-body-decode.md`
- `02-api-input-output/05-zero-null-omitempty.md`
- `02-api-input-output/06-error-handling-validation.md`

你需要掌握：

- 如何设置 `Content-Type` 并返回 JSON。
- 如何从 `http.Request.Body` 解码请求。
- 如何区分字段没传、传了零值、传了 `null`。
- 如何把解析错误转成后端接口里的 400 响应。

真实项目中，创建用户、发布文章、提交订单这些接口都离不开这一阶段。

### 第三阶段：补上生产边界

对应文件：

- `03-production-practice/07-strict-decoder.md`
- `03-production-practice/08-streaming-large-json.md`
- `03-production-practice/09-custom-marshal-unmarshal.md`
- `03-production-practice/10-rawmessage-events.md`
- `03-production-practice/11-number-precision.md`
- `03-production-practice/12-map-interface-dynamic-json.md`

你需要掌握：

- 用 `DisallowUnknownFields` 让请求更严格。
- 用 `Encoder` 和 `Decoder` 避免一次性读入过大数据。
- 用自定义编解码处理时间、状态、脱敏等字段。
- 用 `RawMessage` 延迟解析多类型事件。
- 用 `UseNumber` 和字符串金额避免精度问题。
- 知道什么时候可以用 `map[string]any`，什么时候应该回到结构体。

这一阶段会开始接近生产经验：字段不能乱收，金额不能随便用浮点数，日志不能泄露敏感信息。

### 第四阶段：做一个小项目

对应文件：

- `04-project/13-json-contract-and-version.md`
- `04-project/14-mini-order-api.md`

你需要把前面的知识串起来：

- 设计请求和响应结构体。
- 编写创建订单接口。
- 处理 JSON 错误和业务错误。
- 给后续数据库落库留下安全边界。
- 设计可演进的 JSON 契约。

## 四、真实后端场景示例

贯穿这套路线上下文的真实场景是：你正在做一个 Go 后端服务，需要支持用户资料、文章列表、短链接创建、订单创建、权限变更、审计日志和事件消费。

这些场景会反复出现，因为它们能覆盖后端项目里最常见的 JSON 边界：请求体、响应体、日志、配置、消息事件和数据库 JSON 字段。

## 五、注意点

每篇文章都建议你按下面步骤练：

1. 先复制最小示例运行。
2. 改一个字段名，观察 JSON 输出变化。
3. 故意传错 JSON，观察错误。
4. 把示例改成你自己的业务对象，例如用户、文章、订单。
5. 写下这个知识点在真实项目中能防住什么问题。

Windows PowerShell 常用运行方式：

```powershell
mkdir json-demo
cd json-demo
go mod init json-demo
notepad main.go
go run .
```

Linux/macOS 常用运行方式：

```bash
mkdir json-demo
cd json-demo
go mod init json-demo
nano main.go
go run .
```

这些命令在真实项目里的作用是创建一个独立练习模块，避免污染已有项目。风险是你在已有业务仓库里随手 `go mod init`，可能会生成多余的 `go.mod`。替代方案是在临时目录练习，或者直接使用已有项目的测试包。

### 学习节奏建议

不要急着一天读完所有文章。更建议按“读一篇、跑一个示例、改一个业务对象”的节奏推进。

第一轮只要能理解并运行示例。第二轮把示例改成自己的用户、文章、订单接口。第三轮再关注严格解析、数字精度、事务和迁移这些生产边界。

### 学习边界

学习 `encoding/json` 时，不要把目标停在“会转 JSON”。后端工程师更重要的是知道 JSON 处在系统边界上，外部输入进入系统前要被限制、解析、校验和记录。

同时也不要一开始就追源码。初学阶段先把接口写稳，等你真正遇到性能、兼容或安全问题时，再深入底层行为会更有效。

## 六、常见误区

误区一：只背 API，不写接口。`Marshal` 和 `Unmarshal` 很快能记住，但真正的能力来自请求、响应和错误处理场景。

误区二：把 JSON 校验当成业务校验。JSON 解析成功只说明格式基本正确，不代表用户、库存、金额、权限都合法。

误区三：忽略接口兼容性。字段名随意修改，会影响前端、移动端、第三方脚本和历史数据。

误区四：把数据库安全问题交给 JSON 解决。JSON 解析不能替代参数化查询、事务边界、迁移工具和可回滚方案。

## 七、本节达标标准

学完整套教程后，你应该能做到：

- 能为后端接口设计清晰的请求和响应结构体。
- 能正确使用 `json` 标签维护接口字段名。
- 能安全解析请求体，并返回清晰的错误响应。
- 能识别 `omitempty`、零值、指针和 `null` 的区别。
- 能处理未知字段、重复字段、数字精度等常见坑。
- 能在需要时使用 `Encoder`、`Decoder`、`RawMessage` 和自定义编解码。
- 能知道 JSON 数据进入数据库前还需要参数化查询、事务边界、迁移方案和回滚方案。
