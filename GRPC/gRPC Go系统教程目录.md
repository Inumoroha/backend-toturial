# gRPC Go 系统教程目录

本目录面向想成为 Go 后端工程师的学习者，按“准备阶段 -> 基础语法 -> 服务开发 -> 工程化 -> 项目实战”的顺序组织。

学习建议：

1. 先读 `grpc-go-roadmap.md`，了解整体路线。
2. 再按阶段文件夹顺序学习。
3. 每一节都建议边读边敲代码。
4. 每个阶段完成后，至少完成该阶段的练习任务和完成标准。
5. 第 10 阶段是综合项目，不建议跳过前面阶段直接做。

---

## 阶段目录

### 第1阶段：准备阶段与RPC直觉

- `0_准备阶段总览.md`
- `1_理解RPC与gRPC解决的问题.md`
- `2_安装Go_protoc和代码生成插件.md`
- `3_创建学习工程与目录结构.md`
- `4_调试工具grpcurl与buf初识.md`

### 第2阶段：Protobuf基础与契约设计

- `0_Protobuf阶段总览.md`
- `1_proto3基本语法.md`
- `2_message字段类型与默认值.md`
- `3_字段编号兼容性与版本演进.md`
- `4_service_rpc与go_package.md`
- `5_代码生成与目录管理.md`

### 第3阶段：Unary RPC入门

- `0_Unary_RPC阶段总览.md`
- `1_定义UserService协议.md`
- `2_实现gRPC服务端.md`
- `3_实现gRPC客户端.md`
- `4_context超时与连接复用.md`

### 第4阶段：四种RPC模式

- `0_四种RPC模式总览.md`
- `1_Server_Streaming服务端流.md`
- `2_Client_Streaming客户端流.md`
- `3_Bidirectional_Streaming双向流.md`
- `4_流式RPC常见坑与选择建议.md`

### 第5阶段：错误处理超时取消与Metadata

- `0_错误处理阶段总览.md`
- `1_status_codes错误模型.md`
- `2_deadline超时与取消传播.md`
- `3_metadata_header_trailer.md`
- `4_业务错误映射规范.md`

### 第6阶段：拦截器鉴权日志与链路追踪

- `0_拦截器阶段总览.md`
- `1_Unary_Server_Interceptor.md`
- `2_鉴权拦截器与方法白名单.md`
- `3_Recovery拦截器与panic处理.md`
- `4_Stream_Interceptor与链式组合.md`

### 第7阶段：安全服务治理与生产能力

- `0_生产能力阶段总览.md`
- `1_TLS与mTLS基础.md`
- `2_健康检查与服务反射.md`
- `3_优雅关闭与信号处理.md`
- `4_服务发现负载均衡与重试直觉.md`
- `5_OpenTelemetry观测性入门.md`

### 第8阶段：gRPC-Gateway与HTTP入口

- `0_Gateway阶段总览.md`
- `1_添加HTTP注解与路由设计.md`
- `2_生成Gateway代码与启动HTTP服务.md`
- `3_HTTP错误映射与响应格式.md`
- `4_OpenAPI文档与对外接口管理.md`

### 第9阶段：测试压测与性能优化

- `0_测试压测阶段总览.md`
- `1_bufconn内存集成测试.md`
- `2_错误码Metadata与超时测试.md`
- `3_Streaming测试方法.md`
- `4_ghz压测与性能指标.md`

### 第10阶段：项目实战：订单微服务系统

- `0_订单微服务项目总览.md`
- `1_需求分析与接口边界.md`
- `2_Proto契约设计.md`
- `3_项目结构与公共中间件.md`
- `4_实现UserService.md`
- `5_实现ProductService与库存扣减.md`
- `6_实现OrderService跨服务调用.md`
- `7_订单状态ServerStreaming.md`
- `8_API_Gateway对外HTTP接口.md`
- `9_测试压测与项目收尾.md`

---

## 学习节奏建议

如果你每天学习 1-2 小时，可以这样安排：

- 第 1 周：第 1-2 阶段，重点是工具链和 Protobuf。
- 第 2 周：第 3 阶段，跑通第一个 Unary RPC。
- 第 3 周：第 4 阶段，掌握 streaming。
- 第 4 周：第 5-6 阶段，补齐错误、metadata、拦截器。
- 第 5 周：第 7 阶段，理解生产能力。
- 第 6 周：第 8-9 阶段，掌握 Gateway、测试、压测。
- 第 7-8 周：第 10 阶段，完成订单微服务项目。

---

## 最小闭环

如果你想先快速建立信心，按这个顺序做：

```text
第1阶段工具链
-> 第2阶段 user.proto
-> 第3阶段 UserService server/client
-> 第5阶段错误码和 metadata
-> 第6阶段日志和鉴权拦截器
-> 第9阶段 bufconn 测试
-> 第10阶段项目实战
```