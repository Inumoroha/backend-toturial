# 02 日志与排障：502、503、504 系统排查

## 本节目标

系统掌握 Nginx 代理 Go 服务时最常见的三个错误：

- `502 Bad Gateway`
- `503 Service Temporarily Unavailable`
- `504 Gateway Timeout`

你要形成稳定的排查顺序，而不是看到错误就乱改配置。

## 一、502 Bad Gateway

### 常见原因

- Go 服务没有启动。
- upstream 端口写错。
- Go 服务崩溃。
- Nginx 无法连接后端。
- 后端提前断开连接。
- Docker 容器网络地址写错。

### 排查步骤

1. 看 Nginx error log：

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

2. 直接访问 Go 服务：

```bash
curl http://127.0.0.1:8080/api/ping
```

3. 查看端口：

```bash
ss -lntp | grep 8080
```

4. 查看 Go 服务日志。

5. 检查 Nginx upstream 配置：

```bash
sudo nginx -T | grep -A 10 upstream
```

### 典型 error.log

```text
connect() failed (111: Connection refused) while connecting to upstream
```

解释：Nginx 连后端端口失败。优先检查 Go 服务是否启动、端口是否一致。

## 二、503 Service Temporarily Unavailable

### 常见原因

- upstream 全部不可用。
- 触发限流或连接限制。
- 人为配置了返回 503。
- 上游服务处于不可用状态。

### 排查步骤

1. 检查 access log 中是否有大量 503。
2. 检查是否配置了 `limit_req` 或 `limit_conn`。
3. 检查 upstream 是否全部失败。
4. 检查是否有维护页配置。

## 三、504 Gateway Timeout

### 常见原因

- Go 服务响应太慢。
- 数据库慢查询。
- 外部 API 调用超时。
- Go 服务 goroutine 阻塞。
- `proxy_read_timeout` 太短。

### 排查步骤

1. 看 error log：

```text
upstream timed out while reading response header from upstream
```

2. 看 access log 的耗时：

```text
rt=30.001 urt=30.000
```

3. 直接访问 Go 服务慢接口：

```bash
time curl http://127.0.0.1:8080/api/slow
```

4. 看 Go 服务日志、数据库日志、外部依赖耗时。

5. 判断应该优化业务还是调整超时。

## 四、不要第一反应就加超时时间

看到 504，很多人会直接把超时改成 300 秒。这可能掩盖真实问题。

正确思路：

- 如果接口本来就应该很快，优先优化 Go、数据库、外部调用。
- 如果接口确实是长任务，考虑异步任务、任务队列、轮询结果。
- 只有明确业务需要长连接时，再调整 Nginx 超时。

## 五、排障检查表

### 先确认请求入口

```bash
curl -v http://api.local/api/ping
sudo tail -f /var/log/nginx/access.log
```

### 再确认 Nginx 配置

```bash
sudo nginx -t
sudo nginx -T
```

### 再确认后端服务

```bash
ss -lntp | grep 8080
curl http://127.0.0.1:8080/api/ping
```

### 最后确认业务依赖

- 数据库是否慢。
- Redis 是否慢。
- 外部接口是否慢。
- Go 服务是否有 panic。
- CPU、内存、连接数是否异常。

## 六、本节练习

1. 把 upstream 端口改错，制造 502。
2. 停掉所有 Go 实例，观察错误。
3. 配置限流并压测，观察 503 或 429。
4. 写一个睡眠 40 秒的接口，制造 504。
5. 用日志说明每个错误的根因。

## 七、你应该掌握

学完本节，你应该能稳定排查：

- 502：连接后端失败或后端异常。
- 503：服务不可用、限流或 upstream 不可用。
- 504：后端响应超时。

