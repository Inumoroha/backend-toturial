# 03-Context、超时、连接池与可靠性

## 学习目标

把网络知识落实到后端可靠性设计：超时、取消、连接复用、连接池、重试边界。

## 一、为什么超时是必须的

网络请求可能卡在很多地方：

- DNS 解析。
- TCP 连接。
- TLS 握手。
- 请求写入。
- 服务端处理。
- 响应读取。

没有超时，goroutine 可能长期阻塞，连接和文件描述符也会被占用。

## 二、超时应该分层

一个完整请求链路：

```text
用户 -> API 服务 -> 订单服务 -> 库存服务 -> 数据库
```

如果用户请求总超时是 3 秒，那么 API 服务调用下游不能也设置 3 秒，否则留给自己处理和返回的时间不够。

推荐思路：

- 入口请求有总 deadline。
- 每个下游调用有更短的 timeout。
- 下游调用使用同一个 request context 派生。

## 三、Context 传播

HTTP Handler 中：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
    defer cancel()

    // 用 ctx 调用下游
}
```

当客户端断开连接或请求被取消时，`r.Context()` 会取消。你应该把它传递给下游调用。

## 四、连接池的意义

建立 TCP 和 TLS 连接有成本。

连接池的好处：

- 避免重复握手。
- 降低延迟。
- 减少 CPU 开销。
- 提高吞吐。

但连接池也有风险：

- 上限太小，排队等待连接。
- 上限太大，压垮下游。
- 空闲连接过期后复用失败。

## 五、HTTP Transport 关键参数

```go
transport := &http.Transport{
    MaxIdleConns:          200,
    MaxIdleConnsPerHost:   50,
    MaxConnsPerHost:       100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   3 * time.Second,
    ResponseHeaderTimeout: 2 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```

说明：

- `MaxIdleConns`：所有 host 的最大空闲连接数。
- `MaxIdleConnsPerHost`：每个 host 的最大空闲连接数。
- `MaxConnsPerHost`：每个 host 的最大总连接数。
- `ResponseHeaderTimeout`：等待响应头的最大时间。

## 六、重试要谨慎

重试可以缓解短暂抖动，但也可能放大故障。

适合重试：

- 短暂网络错误。
- 502、503、504。
- 明确幂等的 GET 请求。

不适合盲目重试：

- 创建订单。
- 扣款。
- 写库存。
- 没有幂等键的 POST。

重试必须有：

- 最大次数。
- 超时边界。
- 退避策略。
- 幂等保障。

## 七、故障案例：忘记关闭响应体

现象：

- 服务运行一段时间后请求变慢。
- `ss` 看到大量连接。
- 日志出现 `too many open files`。

错误代码：

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
return nil
```

修复：

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
io.Copy(io.Discard, resp.Body)
```

## 八、练习题

1. 为什么网络请求必须设置超时？
2. 为什么下游调用要使用从入口请求派生的 context？
3. 连接池参数太小或太大分别有什么问题？
4. 为什么重试可能放大故障？

## 九、验收标准

你能设计一个带超时、连接池、context 取消和有限重试的 HTTP 调用封装，并能解释每个参数的作用。

