# 03 日志与排障：请求 ID 与 Nginx/Go 日志关联

## 本节目标

生产排障时，只看 Nginx 日志或只看 Go 日志都不够。你需要用请求 ID 把同一次请求串起来。

## 一、为什么需要请求 ID

一次请求可能经过：

```text
Nginx -> Go API -> Redis -> MySQL -> 第三方 API
```

如果没有请求 ID，你只能靠时间和路径猜测。请求量一大，很难定位。

## 二、Nginx 生成 request_id

Nginx 有变量：

```nginx
$request_id
```

可以传给 Go：

```nginx
proxy_set_header X-Request-ID $request_id;
```

也可以写进日志：

```nginx
log_format api_main 'rid=$request_id '
                    '$remote_addr [$time_local] "$request" '
                    '$status rt=$request_time '
                    'urt=$upstream_response_time '
                    'upstream=$upstream_addr';
```

## 三、Go 服务读取请求 ID

```go
func requestID(r *http.Request) string {
	id := r.Header.Get("X-Request-ID")
	if id == "" {
		id = "missing"
	}
	return id
}
```

在日志中打印：

```go
log.Printf("request_id=%s method=%s path=%s", requestID(r), r.Method, r.URL.Path)
```

这样 Nginx access log 和 Go 应用日志里都会出现同一个 ID。

## 四、客户端自带请求 ID 怎么办

有些客户端会发送：

```text
X-Request-ID: client-generated-id
```

简单做法：由 Nginx 统一覆盖，避免客户端伪造：

```nginx
proxy_set_header X-Request-ID $request_id;
```

如果你希望保留客户端 ID，可以另传一个头：

```nginx
proxy_set_header X-Request-ID $request_id;
proxy_set_header X-Client-Request-ID $http_x_request_id;
```

## 五、完整日志配置

```nginx
log_format api_main 'rid=$request_id '
                    'remote=$remote_addr '
                    'time=[$time_local] '
                    'request="$request" '
                    'status=$status '
                    'bytes=$body_bytes_sent '
                    'rt=$request_time '
                    'uct=$upstream_connect_time '
                    'uht=$upstream_header_time '
                    'urt=$upstream_response_time '
                    'upstream=$upstream_addr';

server {
    listen 80;
    server_name api.local;

    access_log /var/log/nginx/api_access.log api_main;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header X-Request-ID $request_id;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 六、排障示例

Nginx 日志：

```text
rid=abc123 request="GET /api/users HTTP/1.1" status=504 rt=30.001 urt=30.000
```

Go 日志：

```text
request_id=abc123 method=GET path=/api/users db_query_ms=29500
```

结论：Nginx 504 是因为 Go 服务内部数据库查询接近 30 秒。

## 七、本节练习

1. 在 Nginx 日志中加入 `$request_id`。
2. 通过 `X-Request-ID` 传给 Go。
3. Go 服务打印请求 ID。
4. 制造一个慢请求，用请求 ID 同时找到 Nginx 和 Go 日志。
5. 思考生产中是否还要把请求 ID 传给 Redis、MySQL 注释或下游 HTTP 服务。

## 八、你应该掌握

学完本节，你应该能用请求 ID 把一次请求在 Nginx 和 Go 服务中的日志串起来，排障会清楚很多。

