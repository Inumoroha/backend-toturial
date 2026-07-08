# Go Context 系统教程目录

这套教程按照“准备阶段 -> 基础认知 -> 取消控制 -> 超时控制 -> 请求链路 -> Value 使用 -> 组件实践 -> 源码阅读 -> 工程规范 -> 项目实战”的顺序组织。

每个阶段都有独立文件夹，文件名前面的数字表示推荐学习顺序。

---

## 第 0 阶段：学习准备与并发基础

目标：补齐学习 `context` 前必须具备的 Go 并发和后端生命周期直觉。

- `0_学习准备与并发基础总览.md`
- `1_学习环境与练习项目准备.md`
- `2_Goroutine_Channel_Select基础复习.md`
- `3_为什么后端服务需要Context.md`
- `4_本阶段综合练习.md`

---

## 第 1 阶段：Context 基础认知

目标：理解 `context.Context` 接口、根 context、四个核心方法和第一个可取消任务。

- `0_Context基础认知总览.md`
- `1_Context接口的四个方法.md`
- `2_Background与TODO.md`
- `3_Done_Err_Deadline_Value.md`
- `4_第一个Context版任务.md`

---

## 第 2 阶段：取消控制

目标：掌握 `WithCancel`，理解父子 context 取消传播，并能避免 goroutine 泄漏。

- `0_取消控制总览.md`
- `1_WithCancel基础用法.md`
- `2_父子Context与取消传播.md`
- `3_使用Context避免Goroutine泄漏.md`
- `4_取消错误处理与练习.md`

---

## 第 3 阶段：超时与截止时间

目标：掌握 `WithTimeout`、`WithDeadline`，能为 HTTP 和数据库调用设置超时边界。

- `0_超时与截止时间总览.md`
- `1_WithTimeout基础用法.md`
- `2_WithDeadline与超时预算.md`
- `3_HTTP客户端请求超时.md`
- `4_数据库查询超时.md`
- `5_超时错误处理与练习.md`

---

## 第 4 阶段：请求链路传递

目标：掌握从 HTTP handler 到 service、repository、数据库的 context 传递方式。

- `0_请求链路传递总览.md`
- `1_net_http中的Request_Context.md`
- `2_Handler_Service_Repository传递Context.md`
- `3_中间件与RequestID.md`
- `4_不要把Context存进结构体.md`
- `5_请求链路综合练习.md`

---

## 第 5 阶段：Context Value 正确使用

目标：掌握 `context.Value` 的边界，能传递 request id、trace id、认证用户等请求级元数据。

- `0_Context_Value正确使用总览.md`
- `1_WithValue适合传什么.md`
- `2_类型安全的Key与封装函数.md`
- `3_认证信息日志字段与TraceID.md`
- `4_常见滥用案例.md`
- `5_Context_Value综合练习.md`

---

## 第 6 阶段：标准库与常用组件实践

目标：把 context 用到 `net/http`、数据库、信号处理和并发任务中。

- `0_标准库与常用组件实践总览.md`
- `1_net_http服务端与客户端Context.md`
- `2_database_sql与pgx中的Context.md`
- `3_signal_NotifyContext与优雅关闭.md`
- `4_errgroup协同取消.md`
- `5_组件实践综合练习.md`

---

## 第 7 阶段：Context 源码阅读

目标：理解 `emptyCtx`、`cancelCtx`、`timerCtx`、`valueCtx` 和取消传播机制。

- `0_Context源码阅读总览.md`
- `1_Context接口与emptyCtx.md`
- `2_cancelCtx取消实现.md`
- `3_timerCtx超时实现.md`
- `4_valueCtx链式取值.md`
- `5_propagateCancel取消传播.md`
- `6_源码阅读总结.md`

---

## 第 8 阶段：工程规范与故障排查

目标：形成可执行的团队规范，能设计超时预算，能排查 context 相关问题。

- `0_工程规范与故障排查总览.md`
- `1_Context代码规范.md`
- `2_超时预算设计.md`
- `3_错误处理日志与监控.md`
- `4_Context相关测试.md`
- `5_常见问题排查清单.md`

---

## 第 9 阶段：项目实战

目标：通过 HTTP 聚合服务、Worker 优雅关闭、数据库查询超时保护三个项目完成闭环。

- `0_项目实战总览.md`
- `1_项目一_HTTP聚合服务需求与结构.md`
- `2_项目一_errgroup并发调用与取消.md`
- `3_项目一_超时错误与RequestID日志.md`
- `4_项目二_Worker服务优雅关闭.md`
- `5_项目三_数据库查询超时保护.md`
- `6_综合检查清单.md`

---

## 推荐学习方式

建议每学一个阶段都做三件事：

1. 读完该阶段所有 Markdown。
2. 手敲其中至少 2 个代码示例。
3. 完成本阶段综合练习或检查清单。

如果某个阶段读起来吃力，优先回到第 0 阶段复习 goroutine、channel 和 select。`context` 的难点往往不在 API，而在并发和请求生命周期。

---

## 最终完成标准

当你能独立完成下面三件事时，就说明这套教程的主线已经学通：

- 写出一个支持请求总超时和下游调用超时的 HTTP 接口。
- 写出一个收到退出信号后能优雅停止的 worker 服务。
- 写出一个 handler、service、repository 全链路传递 context 的数据库查询接口。
