# 02 负载均衡：失败处理、重试与会话策略

## 本节目标

学习当某个 Go 实例挂掉时，Nginx 会怎么处理，以及后端服务如何设计得更适合负载均衡。

## 一、模拟后端故障

保持三个实例运行：

```text
8080
8081
8082
```

然后停止其中一个，比如 `8081`。

继续请求：

```bash
for i in {1..20}; do curl -s -o /dev/null -w "%{http_code}\n" http://api.local/api/ping; done
```

观察：

- 是否出现 502。
- error log 中是否有连接失败。
- 请求是否继续落到其他实例。

查看日志：

```bash
sudo tail -f /var/log/nginx/error.log
```

## 二、max_fails 与 fail_timeout

```nginx
upstream go_api {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8081 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=10s;
}
```

含义：

- `max_fails=3`：在指定时间窗口内失败 3 次后，认为该后端暂时不可用。
- `fail_timeout=10s`：失败统计窗口，也是暂时不可用的时间。

这不是完整主动健康检查，而是被动失败判断。

## 三、proxy_next_upstream

```nginx
location / {
    proxy_pass http://go_api;
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 3;
}
```

含义：

- 如果连接错误、超时、或遇到指定状态码，可以尝试下一个 upstream。
- `proxy_next_upstream_tries 3` 限制最多尝试次数。

注意：对非幂等请求要小心，比如支付、创建订单、扣库存。POST 请求重试可能造成重复写入。业务层必须有幂等设计。

## 四、会话与无状态服务

负载均衡下，用户的两次请求可能落到不同实例。如果你的 Go 服务把登录状态存在进程内存里，就会出问题。

不推荐：

```text
用户登录 -> session 存在 8080 进程内存
下一次请求 -> 被转发到 8081
8081 找不到 session
```

推荐：

- 使用 JWT。
- session 存 Redis。
- 登录态存在共享数据库或缓存。
- 服务实例尽量无状态。

## 五、健康检查思路

Go 服务建议提供健康检查接口：

```go
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ok"))
})
```

Nginx 开源版主要依赖被动检查。生产中如果需要主动健康检查，可以考虑：

- Nginx Plus。
- Kubernetes readinessProbe。
- 云负载均衡健康检查。
- Consul、Traefik、Envoy、Kong 等组件。

## 六、完整示例

```nginx
upstream go_api {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8081 max_fails=3 fail_timeout=10s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=10s;
}

server {
    listen 80;
    server_name api.local;

    location / {
        proxy_pass http://go_api;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 3;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 七、本节练习

1. 停掉一个 Go 实例，观察 Nginx 行为。
2. 配置 `max_fails` 和 `fail_timeout`。
3. 配置 `proxy_next_upstream`。
4. 写一个 `/health` 接口。
5. 思考你的 Go 服务是否可以无状态运行。

## 八、你应该掌握

学完本节，你应该知道：

- Nginx 如何处理失败 upstream。
- 为什么重试可能带来重复写入问题。
- 为什么负载均衡要求后端尽量无状态。
- 健康检查在生产部署中为什么重要。

