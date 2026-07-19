# 阶段 0：Go 后端与 Linux 基础

这一阶段的目标是准备“被监控对象”。如果服务本身没有稳定的请求语义、状态码、超时和日志，后续的指标只能把混乱放大。

## 学习目标

完成后应该能够：

- 写一个有路由、错误处理、超时和优雅退出的 Go HTTP 服务。
- 用单元测试和 `httptest` 验证请求行为。
- 用 `go test -race`、benchmark 和 pprof 建立基础性能证据。
- 使用 Linux/容器命令判断端口、进程、DNS 和网络连通性。
- 区分应用返回错误、网络不可达和进程已经退出。

## 教程顺序

1. [实现可诊断的 Go HTTP 服务](01-go-http-service.md)
2. [Linux、Docker 网络与故障排查](02-linux-network-debug.md)
3. [可观测性基础与 SLO](03-observability-basics.md)

## 两周任务计划

### 第 1 周：服务和测试

- 创建 module 和目录结构。
- 实现 `GET /hello`、`POST /orders`、`GET /healthz`。
- 添加 context 超时、结构化日志和优雅退出。
- 用 `go test ./...` 和 `go test -race ./...` 验证。
- 写一份请求状态码和错误码表。

### 第 2 周：系统和观测思维

- 使用 `ss`/`netstat`、`curl -v`、`dig`/`nslookup`、`docker inspect` 检查网络。
- 给服务增加 pprof 和可配置的延迟/错误注入。
- 画出“客户端 -> 服务 -> 依赖”的请求路径。
- 使用 RED 和 USE 方法为服务列出候选指标。

## 阶段交付物

- 一个可启动的 Go 服务。
- 单元测试、race 检测和 benchmark 输出。
- 一份 `README.md`，包含启动、测试、故障注入和清理命令。
- 一张请求路径图。
- 一份指标设计草案，下一阶段会把它实现成 Prometheus 指标。

## 验收

删除二进制和容器后，能在 10 分钟内重新启动服务；人为让模拟依赖超时后，能说明请求状态码、日志和服务进程分别发生了什么变化。

