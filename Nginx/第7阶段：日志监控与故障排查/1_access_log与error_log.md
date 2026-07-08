# 1. access_log 与 error_log

本节目标：理解 Nginx 两类核心日志，并能用它们判断请求是否进入 Nginx、是否转发到 Go、错误发生在哪一层。

---

## 一、两类日志的职责

### access log

记录每个请求的访问结果。

常见信息：

- 客户端 IP。
- 请求时间。
- 请求方法和路径。
- 状态码。
- 响应大小。
- User-Agent。
- 请求耗时。
- upstream 地址和耗时。

### error log

记录 Nginx 运行和处理请求时的错误。

常见信息：

- 配置加载错误。
- 文件权限错误。
- upstream 连接失败。
- upstream 超时。
- 请求体过大。

---

## 二、默认日志位置

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

---

## 三、一次请求怎么观察

终端 1：

```bash
sudo tail -f /var/log/nginx/access.log
```

终端 2：

```bash
curl -H "Host: api.local" http://127.0.0.1/api/ping
```

如果 access log 出现新行，说明请求进入了 Nginx。

如果 Go 服务也打印日志，说明请求进入了 Go。

如果 access log 有记录但 Go 没日志，说明请求可能被 Nginx 直接处理、限流、拒绝，或匹配到了错误 location。

---

## 四、自定义日志格式

对 Go 后端非常有用的日志格式：

```nginx
log_format api_main '$remote_addr [$time_local] "$request" '
                    'status=$status bytes=$body_bytes_sent '
                    'rt=$request_time '
                    'uct=$upstream_connect_time '
                    'uht=$upstream_header_time '
                    'urt=$upstream_response_time '
                    'upstream=$upstream_addr';
```

使用：

```nginx
access_log /var/log/nginx/api_access.log api_main;
```

字段解释：

- `rt`：整个请求耗时。
- `uct`：连接 upstream 耗时。
- `uht`：收到 upstream 响应头耗时。
- `urt`：upstream 总响应耗时。
- `upstream`：实际后端地址。

---

## 五、通过日志判断慢在哪里

示例：

```text
rt=5.002 urt=5.001
```

说明总耗时几乎都在 Go upstream。

示例：

```text
uct=1.500
```

说明连接后端就很慢，可能是后端压力大或连接队列问题。

示例：

```text
upstream=127.0.0.1:8081
```

说明这次请求实际打到了 `8081`。

---

## 六、error_log 常见信息

### 连接失败

```text
connect() failed (111: Connection refused) while connecting to upstream
```

常见原因：

- Go 服务没启动。
- 端口写错。
- upstream 地址错。

### upstream 超时

```text
upstream timed out while reading response header from upstream
```

常见原因：

- Go 接口太慢。
- 数据库慢查询。
- `proxy_read_timeout` 触发。

### 请求体过大

```text
client intended to send too large body
```

常见原因：

- 超过 `client_max_body_size`。

---

## 七、本节练习

1. 配置自定义 `api_main` 日志格式。
2. 请求 `/api/ping`，观察 upstream 地址。
3. 请求 `/api/slow?seconds=3`，观察耗时。
4. 停掉 Go 服务，观察 error log。
5. 故意上传大文件，观察 error log。

---

## 八、本节复盘

请确认你能回答：

1. access log 和 error log 分别看什么？
2. 如何判断请求有没有进入 Nginx？
3. 如何判断请求有没有进入 Go？
4. `upstream_response_time` 大说明什么？
5. `Connection refused` 通常说明什么？

