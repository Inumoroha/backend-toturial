# 03 负载均衡：幂等、优雅重启与发布风险

## 本节目标

负载均衡不只是把请求分给多个实例。你还要考虑：

- 请求失败后能不能重试。
- Go 服务发布时正在处理的请求怎么办。
- 多实例之间如何保持行为一致。

## 一、什么是幂等

幂等指同一个操作执行一次和执行多次，结果一致。

通常幂等：

- 查询用户信息。
- 查询商品详情。
- 获取配置。

通常非幂等：

- 创建订单。
- 支付扣款。
- 发优惠券。
- 扣库存。

## 二、Nginx 重试的风险

配置：

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 3;
```

如果请求是：

```text
POST /api/orders
```

Nginx 第一次转发给 `8080`，Go 服务已经创建订单，但响应时连接断了。Nginx 可能重试到 `8081`，结果订单被创建两次。

所以业务写接口必须考虑幂等。

## 三、Go 业务幂等思路

常见做法：

- 客户端提交 `Idempotency-Key`。
- 服务端用唯一请求号防重复。
- 数据库使用唯一索引。
- 支付、订单等关键操作使用业务流水号。

示例请求头：

```text
Idempotency-Key: order-create-20260705-0001
```

Go 服务收到后，先查这个 key 是否处理过，再决定是否执行。

## 四、Nginx 对不同方法的重试建议

保守策略：

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 2;
```

并且对关键 POST 接口少依赖 Nginx 重试，更多依赖业务幂等。

如果要更细，可以给只读接口和写接口拆不同 location：

```nginx
location /api/query/ {
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 3;
    proxy_pass http://go_api;
}

location /api/order/ {
    proxy_next_upstream error timeout;
    proxy_next_upstream_tries 1;
    proxy_pass http://go_api;
}
```

## 五、Go 服务优雅关闭

发布时如果直接 kill Go 进程，正在处理的请求可能中断。

Go 可以使用 `http.Server.Shutdown` 做优雅关闭：

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() {
	if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatal(err)
	}
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
	log.Fatal(err)
}
```

这样进程收到退出信号后，会停止接收新连接，并等待已有请求完成。

## 六、systemd 配合优雅关闭

```ini
[Service]
ExecStart=/opt/myapp/myapp
Restart=always
KillSignal=SIGTERM
TimeoutStopSec=15
```

`TimeoutStopSec` 应该大于 Go 服务优雅关闭的等待时间。

## 七、发布顺序建议

单机多实例时：

1. 从 upstream 临时移除一个实例或停止一个实例。
2. 发布该实例。
3. 健康检查通过后加入流量。
4. 继续发布下一个实例。

容器或 Kubernetes 环境中，则通常由编排系统负责滚动发布。

## 八、本节练习

1. 写一个非幂等模拟接口，例如计数器加一。
2. 配置 Nginx 重试并观察潜在重复执行风险。
3. 给请求加 `Idempotency-Key`，用 Go 代码防重复。
4. 给 Go 服务加入优雅关闭。
5. 模拟发布时中断请求，观察差异。

## 九、你应该掌握

学完本节，你应该知道：负载均衡和重试会改变请求执行模型。真正可靠的后端系统，需要 Nginx 配置和 Go 业务幂等一起设计。

