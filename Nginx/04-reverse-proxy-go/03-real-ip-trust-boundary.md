# 03 反向代理 Go 服务：真实 IP 与信任边界

## 本节目标

很多教程会告诉你配置：

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

但没有告诉你：这个头可以被客户端伪造。Go 后端工程师必须理解真实 IP 的信任边界。

## 一、RemoteAddr 为什么不够

Go 中：

```go
r.RemoteAddr
```

表示与 Go 服务直接建立连接的对端地址。

如果请求链路是：

```text
Client -> Nginx -> Go
```

那么 Go 看到的 `RemoteAddr` 通常是 Nginx 的地址，不是客户端真实 IP。

## 二、X-Forwarded-For 是什么

`X-Forwarded-For` 通常是一个 IP 列表：

```text
client_ip, proxy1_ip, proxy2_ip
```

Nginx 使用：

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

含义是：把已有的 `X-Forwarded-For` 加上当前客户端 IP。

## 三、为什么不能直接相信请求头

客户端可以自己发送：

```bash
curl -H "X-Forwarded-For: 1.2.3.4" http://api.local/api/headers
```

如果 Nginx 直接追加，Go 可能看到：

```text
1.2.3.4, 127.0.0.1
```

这时第一个 IP 不一定可信。

## 四、简单环境的推荐做法

如果你的链路只有：

```text
Client -> Nginx -> Go
```

并且 Go 服务不对公网开放，只允许 Nginx 访问，那么 Go 可以信任 Nginx 设置的：

```text
X-Real-IP
```

Nginx 配置：

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Go 获取：

```go
func clientIP(r *http.Request) string {
	if ip := r.Header.Get("X-Real-IP"); ip != "" {
		return ip
	}
	return r.RemoteAddr
}
```

前提：Go 服务不能被客户端绕过 Nginx 直接访问。

## 五、多层代理的做法

如果链路是：

```text
Client -> Cloud Load Balancer -> Nginx -> Go
```

Nginx 的 `$remote_addr` 可能是云负载均衡的 IP，而不是用户 IP。

这时要使用 Nginx realip 模块：

```nginx
set_real_ip_from 10.0.0.0/8;
set_real_ip_from 192.168.0.0/16;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

含义：

- 只信任来自指定代理网段的转发头。
- 从 `X-Forwarded-For` 中递归找真实客户端 IP。

注意：`set_real_ip_from` 必须写可信代理的 IP 或网段，不能随便写 `0.0.0.0/0`。

## 六、Go 服务的安全建议

### 不要让 Go 服务暴露公网

如果 Go 端口 `8080` 暴露公网，攻击者可以绕过 Nginx：

```text
Client -> Go :8080
```

这样 Nginx 的限流、HTTPS、安全头、真实 IP 逻辑都可能失效。

建议：

- Go 服务监听 `127.0.0.1:8080`，只允许本机 Nginx 访问。
- 或通过防火墙只允许 Nginx 所在机器访问。
- 容器环境中只暴露 Nginx 端口。

### 对敏感风控不要只依赖 IP

IP 可以变化，也可能经过代理。登录保护、风控、限流应结合：

- 用户 ID。
- 设备信息。
- Token。
- 行为频率。
- 业务上下文。

## 七、验证实验

1. 在 Go 服务中打印 `RemoteAddr`、`X-Real-IP`、`X-Forwarded-For`。
2. 正常请求：

```bash
curl http://api.local/api/headers
```

3. 伪造请求头：

```bash
curl -H "X-Forwarded-For: 1.2.3.4" http://api.local/api/headers
```

4. 观察 Go 收到的头有什么变化。
5. 思考你的服务应该信任哪个值。

## 八、你应该掌握

学完本节，你应该知道：

- `RemoteAddr`、`X-Real-IP`、`X-Forwarded-For` 的区别。
- 客户端可以伪造代理头。
- 真实 IP 的可信性来自网络边界，而不是头字段本身。
- Go 服务最好不要绕过 Nginx 暴露公网。

