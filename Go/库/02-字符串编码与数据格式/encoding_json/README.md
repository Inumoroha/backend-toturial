# Go encoding/json 后端实践教程

这套教程面向正在学习 Go 后端开发的学习者，目标不是背标准库文档，而是把 `encoding/json` 放进真实后端项目的请求、响应、日志、配置、事件和数据落库流程里学习。

建议阅读顺序：

1. [00. 学习路线](00-learning-route/00-learning-route.md)
2. [01. 把结构体转换成 JSON 响应](01-basics/01-marshal-unmarshal.md)
3. [02. 用结构体标签稳定接口字段](01-basics/02-struct-tags.md)
4. [03. 在 HTTP 接口中返回 JSON](02-api-input-output/03-http-response-json.md)
5. [04. 解析请求体并做第一层校验](02-api-input-output/04-request-body-decode.md)
6. [05. 区分零值、空值和未传字段](02-api-input-output/05-zero-null-omitempty.md)
7. [06. 处理 JSON 解析错误](02-api-input-output/06-error-handling-validation.md)
8. [07. 开启严格解析，拦住未知字段](03-production-practice/07-strict-decoder.md)
9. [08. 用 Encoder 和 Decoder 处理流式 JSON](03-production-practice/08-streaming-large-json.md)
10. [09. 自定义 JSON 编解码](03-production-practice/09-custom-marshal-unmarshal.md)
11. [10. 用 RawMessage 处理可扩展事件](03-production-practice/10-rawmessage-events.md)
12. [11. 处理数字精度和金额字段](03-production-practice/11-number-precision.md)
13. [12. 处理动态 JSON 和 map 场景](03-production-practice/12-map-interface-dynamic-json.md)
14. [13. 管理接口契约和版本演进](04-project/13-json-contract-and-version.md)
15. [14. 小项目：订单接口中的 JSON 实战](04-project/14-mini-order-api.md)

本教程示例基于本机 `go1.26.0` 的标准库能力编写。`encoding/json` 是稳定标准库，但项目中真正容易出问题的地方通常不在语法，而在接口契约、错误处理、字段演进、数字精度和生产安全边界。

