# 4. Transport 与连接池

本节目标：理解 `http.Transport` 的作用，以及为什么复用 Client 能带来连接复用。

---

## 一、Client 和 Transport 的关系

`http.Client` 负责高层请求流程，底层连接管理主要由 `http.Transport` 完成。

简化理解：

```text
http.Client
-> http.Transport
-> TCP/TLS connection
```

默认 Client 也有 Transport，但生产项目中有时会显式配置。

---

## 二、基础 Transport 配置

```go
transport := &http.Transport{
	MaxIdleConns:        100,
	MaxIdleConnsPerHost: 10,
	IdleConnTimeout:     90 * time.Second,
}

client := &http.Client{
	Timeout:   5 * time.Second,
	Transport: transport,
}
```

字段含义：

- `MaxIdleConns`：所有 host 的最大空闲连接数。
- `MaxIdleConnsPerHost`：每个 host 的最大空闲连接数。
- `IdleConnTimeout`：空闲连接保留多久。

---

## 三、为什么连接池重要

每次请求都重新建连接会有成本：

- TCP 握手。
- TLS 握手。
- 慢启动。

连接复用可以降低延迟和资源消耗。

这也是为什么不要每次请求都新建 Client。

---

## 四、什么时候需要调 Transport

普通项目先使用默认配置通常可以。

需要关注 Transport 的场景：

- 高频调用同一个第三方服务。
- 内部服务之间 QPS 较高。
- 出现连接不足或连接过多。
- 需要配置代理。
- 需要自定义 TLS。

不要一开始就乱调参数。先理解默认行为，再基于压测和监控调整。

---

## 五、本节检查点

请确认你能回答：

- `http.Transport` 主要负责什么？
- 为什么复用 Client 有利于连接复用？
- `MaxIdleConnsPerHost` 大致控制什么？
- 什么时候才需要自定义 Transport？

