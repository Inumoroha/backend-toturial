# 04-HTTP 进阶：Cookie、缓存、HTTP/2 与 HTTP/3

## 学习目标

补齐 HTTP 中后端工程师必须理解的进阶知识：Cookie、Session、缓存控制、HTTP/1.1 队头阻塞、HTTP/2 多路复用、HTTP/3 与 QUIC。

## 一、HTTP 无状态是什么意思

HTTP 无状态表示：协议本身不会记住上一次请求是谁发的、做过什么。

例如客户端连续请求：

```text
GET /cart
POST /cart/items
GET /cart
```

HTTP 协议本身不会自动知道这三个请求来自同一个用户。

后端需要通过 Cookie、Token、Session 等机制建立用户状态。

## 二、Cookie

Cookie 是浏览器保存并随请求自动发送的一小段数据。

服务端设置：

```http
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

浏览器后续请求：

```http
Cookie: session_id=abc123
```

常见属性：

- `HttpOnly`：禁止 JavaScript 读取，降低 XSS 窃取风险。
- `Secure`：只在 HTTPS 下发送。
- `SameSite`：降低 CSRF 风险。
- `Path`：限制 Cookie 发送路径。
- `Max-Age` / `Expires`：控制过期时间。

## 三、Session

Session 通常指服务端保存的会话状态。

流程：

1. 用户登录成功。
2. 服务端生成 `session_id`。
3. 服务端把 `session_id` 写入 Cookie。
4. 后续请求带上 Cookie。
5. 服务端用 `session_id` 查询用户状态。

Session 可以存储在：

- 进程内存。
- Redis。
- 数据库。

生产环境通常不建议只存在单机内存，因为多实例负载均衡会导致状态不一致。

## 四、Token 与 Cookie 的关系

Token 是认证凭据，Cookie 是浏览器保存和发送数据的一种机制。

Token 可以放在：

```http
Authorization: Bearer xxx
```

也可以放在 Cookie 中。

不要把二者混为一谈：

- Cookie 是传输和存储机制。
- Token 是认证内容。

## 五、HTTP 缓存

缓存可以减少服务端压力和网络传输。

常见响应头：

```http
Cache-Control: max-age=60
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

### 强缓存

如果 `Cache-Control: max-age=60` 未过期，浏览器可以直接使用本地缓存，不请求服务端。

### 协商缓存

缓存过期后，浏览器向服务端确认资源是否变化：

```http
If-None-Match: "abc123"
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
```

如果未变化，服务端返回：

```http
304 Not Modified
```

## 六、后端接口如何设置缓存

适合缓存：

- 静态资源。
- 公开配置。
- 不频繁变化的列表。
- 图片、CSS、JS。

不适合随便缓存：

- 用户隐私数据。
- 订单状态。
- 余额。
- 强一致性要求的数据。

常用策略：

```http
Cache-Control: no-store
```

表示不要缓存，适合敏感接口。

```http
Cache-Control: public, max-age=31536000, immutable
```

适合带 hash 的静态资源。

## 七、HTTP/1.1 队头阻塞

HTTP/1.1 虽然支持长连接，但同一个连接上的请求通常按顺序处理响应。

如果前一个响应很慢，后面的响应会被挡住。这就是 HTTP/1.1 层面的队头阻塞。

浏览器通常会对同一域名开多个 TCP 连接来缓解。

## 八、HTTP/2 多路复用

HTTP/2 把请求和响应拆成二进制帧，在同一个 TCP 连接上并发传输多个 Stream。

优点：

- 多路复用。
- Header 压缩。
- 更少 TCP 连接。
- 更适合 gRPC。

但 HTTP/2 仍然运行在 TCP 上。如果 TCP 层丢包，同一个 TCP 连接上的所有 Stream 都可能受影响，这是 TCP 层队头阻塞。

## 九、HTTP/3 与 QUIC

HTTP/3 基于 QUIC，QUIC 基于 UDP。

QUIC 提供：

- 类似 TCP 的可靠传输。
- TLS 1.3 集成。
- 更快连接建立。
- 多路复用时减少 TCP 层队头阻塞影响。
- 连接迁移，例如移动网络切换。

为什么 QUIC 用 UDP？

因为 TCP 在操作系统内核中演进慢，QUIC 在用户态基于 UDP 实现，更容易迭代。

## 十、Go 后端关联

Go 标准库 HTTP Server 默认支持 HTTP/1.1；启用 TLS 时，现代 Go 版本通常可自动支持 HTTP/2。

gRPC 依赖 HTTP/2，这也是为什么 gRPC 天然支持流式调用和多路复用。

后端工程关注点：

- 对外 API 不一定需要 HTTP/2，但需要理解代理和网关是否支持。
- gRPC 需要关注 HTTP/2 连接、流、超时和流控。
- HTTP/3 目前更多出现在边缘网关、CDN、浏览器访问场景。

## 十一、实验：观察缓存

请求一个带缓存头的资源：

```bash
curl -I https://example.com
```

观察：

- `Cache-Control`
- `ETag`
- `Last-Modified`

如果有 ETag，可以尝试：

```bash
curl -H 'If-None-Match: "你的ETag"' -I https://example.com
```

## 十二、实验：观察 HTTP/2

```bash
curl -v --http2 https://example.com
```

关注：

- ALPN 是否协商到 `h2`。
- TLS 是否成功。
- 响应协议版本。

## 十三、常见误区

- HTTP 无状态不代表后端不能保存状态，而是协议本身不保存。
- Cookie 不等于 Session。
- Token 不一定必须放在 LocalStorage，也可以放在 Cookie。
- HTTP/2 解决了 HTTP/1.1 应用层队头阻塞，但没有消除 TCP 丢包带来的影响。
- HTTP/3 不是“HTTP/2 的小升级”，底层传输已经从 TCP 换成 QUIC/UDP。

## 十四、练习题

1. HTTP 无状态是什么意思？
2. Cookie 和 Session 有什么关系？
3. `HttpOnly`、`Secure`、`SameSite` 分别解决什么问题？
4. 强缓存和协商缓存有什么区别？
5. HTTP/2 多路复用为什么适合 gRPC？
6. HTTP/3 为什么使用 QUIC？

## 十五、验收标准

你能解释 Cookie、Session、Token、缓存控制、HTTP/1.1 队头阻塞、HTTP/2 多路复用和 HTTP/3/QUIC 的核心差异，并能用 `curl -I`、`curl -v --http2` 做基础观察。

