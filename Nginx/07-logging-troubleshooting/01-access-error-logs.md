# 01 日志与排障：access.log、error.log 与日志格式

## 本节目标

掌握 Nginx 两类核心日志：

- `access.log`：每个请求的访问记录。
- `error.log`：Nginx 运行和代理过程中的错误信息。

生产排障时，日志是第一入口。

## 一、日志默认位置

常见路径：

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

查看：

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

## 二、access.log 看什么

一条访问日志通常包含：

- 客户端 IP。
- 请求时间。
- 请求方法和路径。
- 状态码。
- 响应大小。
- Referer。
- User-Agent。

示例：

```text
127.0.0.1 - - [05/Jul/2026:18:00:00 +0800] "GET /api/ping HTTP/1.1" 200 18 "-" "curl/8.0"
```

## 三、自定义日志格式

对 Go 后端排障，建议记录 upstream 信息。

```nginx
log_format api_main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" '
                    'rt=$request_time '
                    'uct=$upstream_connect_time '
                    'uht=$upstream_header_time '
                    'urt=$upstream_response_time '
                    'upstream=$upstream_addr';

access_log /var/log/nginx/api_access.log api_main;
```

字段含义：

- `request_time`：Nginx 从收到请求到响应完成的总时间。
- `upstream_connect_time`：连接后端耗时。
- `upstream_header_time`：收到后端响应头耗时。
- `upstream_response_time`：后端整体响应耗时。
- `upstream_addr`：实际请求的后端地址。

## 四、Go 服务慢响应分析

如果日志中：

```text
rt=5.002 urt=5.001
```

说明总耗时主要来自后端 Go 服务。

如果：

```text
uct=2.000
```

说明连接后端就很慢，可能是端口、网络、连接队列、服务压力问题。

## 五、error.log 看什么

常见错误：

```text
connect() failed (111: Connection refused) while connecting to upstream
```

通常表示后端服务没启动或端口不对。

```text
upstream timed out while reading response header from upstream
```

通常表示 Go 服务响应太慢，触发超时。

```text
client intended to send too large body
```

表示请求体超过 `client_max_body_size`。

## 六、日志排障基本流程

1. 先看 `access.log`，确认 Nginx 是否收到请求。
2. 看状态码，是 2xx、4xx 还是 5xx。
3. 看 upstream 地址，确认请求打到了哪个 Go 实例。
4. 看耗时字段，判断慢在 Nginx 还是 Go。
5. 看 `error.log`，找连接失败、超时、权限等信息。
6. 再看 Go 服务日志，确认业务处理结果。

## 七、本节练习

1. 配置自定义日志格式。
2. 请求 `/api/ping`，观察日志。
3. 停掉 Go 服务，观察 `error.log`。
4. 写一个慢接口，观察 `request_time` 和 `upstream_response_time`。
5. 修改请求体大小限制，观察大请求错误。

## 八、你应该掌握

学完本节，你应该能根据日志判断：

- 请求是否进入 Nginx。
- 请求转发到了哪个 Go 实例。
- 慢请求主要慢在后端还是 Nginx。
- 常见代理错误对应什么原因。

