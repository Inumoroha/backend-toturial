# 2. 数据模型、API 与 key 设计

这一节设计短链接项目的基础结构。

写代码之前，先把数据模型、接口和 Redis key 想清楚。否则后面很容易出现字段混乱、key 命名失控、缓存难以失效的问题。

学完这一节后，你应该能够：

- 设计短链接数据库表。
- 设计核心 API 请求和响应。
- 设计 Redis key 命名。
- 为不同 key 设置合理 TTL。
- 区分详情缓存、计数器、限流 key 和空值缓存。

---

## 一、数据库表设计

短链接主表可以这样设计：

```sql
CREATE TABLE short_links (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(32) NOT NULL UNIQUE,
    original_url TEXT NOT NULL,
    title VARCHAR(255) NOT NULL DEFAULT '',
    user_id BIGINT NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'active',
    expires_at TIMESTAMP NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_short_links_user_id ON short_links(user_id);
CREATE INDEX idx_short_links_created_at ON short_links(created_at);
```

如果使用 MySQL，可以把 `BIGSERIAL` 换成 `BIGINT AUTO_INCREMENT`。

字段说明：

| 字段 | 说明 |
| --- | --- |
| `code` | 短码，比如 `aB9xK2` |
| `original_url` | 原始链接 |
| `title` | 标题，可选 |
| `user_id` | 创建者 |
| `status` | `active`、`disabled` |
| `expires_at` | 过期时间 |

---

## 二、领域模型

Go 中可以先定义一个简单结构：

```go
type ShortLink struct {
    ID          int64      `json:"id"`
    Code        string     `json:"code"`
    OriginalURL string     `json:"original_url"`
    Title       string     `json:"title"`
    UserID      int64      `json:"user_id"`
    Status      string     `json:"status"`
    ExpiresAt   *time.Time `json:"expires_at,omitempty"`
    CreatedAt   time.Time  `json:"created_at"`
    UpdatedAt   time.Time  `json:"updated_at"`
}
```

缓存时不一定要保存所有字段。

跳转链路通常只需要：

- 短码。
- 原始 URL。
- 状态。
- 过期时间。

---

## 三、API 设计

### 1. 创建短链接

```text
POST /links
```

请求：

```json
{
  "original_url": "https://example.com/articles/100",
  "title": "Redis 教程",
  "expires_at": "2026-12-31T23:59:59Z"
}
```

响应：

```json
{
  "code": "aB9xK2",
  "short_url": "https://s.example.com/aB9xK2",
  "original_url": "https://example.com/articles/100"
}
```

### 2. 跳转短链接

```text
GET /:code
```

成功时返回 302。

失败时返回 404 或 410。

### 3. 查询详情

```text
GET /links/:code
```

响应：

```json
{
  "code": "aB9xK2",
  "original_url": "https://example.com/articles/100",
  "title": "Redis 教程",
  "status": "active",
  "expires_at": null
}
```

### 4. 更新短链接

```text
PATCH /links/:code
```

请求：

```json
{
  "original_url": "https://example.com/articles/101",
  "title": "Redis 项目教程",
  "status": "active"
}
```

### 5. 查询统计

```text
GET /links/:code/stats
```

响应：

```json
{
  "code": "aB9xK2",
  "visit_count": 12345
}
```

---

## 四、Redis key 设计

路线图推荐 key：

```text
shortlink:detail:{code}
shortlink:visit_count:{code}
shortlink:create_limit:{user_id}
shortlink:not_found:{code}
```

具体示例：

```text
shortlink:detail:aB9xK2
shortlink:visit_count:aB9xK2
shortlink:create_limit:1001
shortlink:not_found:unknownCode
```

命名建议：

- 使用业务名前缀：`shortlink`。
- 使用语义分段：`detail`、`visit_count`。
- 把变量放在最后。
- 不要让 key 太长。
- 不要把完整 URL 放进 key。

---

## 五、key 类型和 TTL

| key | 类型 | TTL | 说明 |
| --- | --- | --- | --- |
| `shortlink:detail:{code}` | String JSON | 5 分钟到 1 小时 | 详情缓存 |
| `shortlink:visit_count:{code}` | String Integer | 可不设或长 TTL | 访问计数 |
| `shortlink:create_limit:{user_id}` | String Integer | 60 秒 | 创建限流 |
| `shortlink:not_found:{code}` | String | 30 秒到 2 分钟 | 空值缓存 |

详情缓存 TTL 可以根据业务调整。

空值缓存 TTL 要短。

限流 key 必须设置 TTL。

访问计数如果后续会同步到数据库，可以结合业务设置更长 TTL。

---

## 六、详情缓存 JSON

推荐缓存结构：

```json
{
  "code": "aB9xK2",
  "original_url": "https://example.com/articles/100",
  "status": "active",
  "expires_at": null
}
```

Go 结构：

```go
type CachedShortLink struct {
    Code        string     `json:"code"`
    OriginalURL string     `json:"original_url"`
    Status      string     `json:"status"`
    ExpiresAt   *time.Time `json:"expires_at,omitempty"`
}
```

不要把特别大的字段放进缓存。

缓存要服务高频读链路。

---

## 七、短码设计

短码可以使用：

- 随机字符串。
- 数据库 ID 转 Base62。
- 雪花 ID 后转 Base62。
- 原 URL hash 后截断。

学习项目推荐先用随机字符串。

示例：

```go
const alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func GenerateCode(n int) string {
    b := make([]byte, n)
    for i := range b {
        b[i] = alphabet[rand.Intn(len(alphabet))]
    }
    return string(b)
}
```

生产中要处理冲突。

数据库的 `UNIQUE(code)` 是最后防线。

---

## 八、缓存失效规则

需要删除缓存的场景：

- 更新原始 URL。
- 禁用短链接。
- 修改过期时间。
- 删除短链接。

需要删除的 key：

```text
shortlink:detail:{code}
shortlink:not_found:{code}
```

通常不删除访问计数。

计数是统计数据，不是详情缓存。

---

## 九、常见错误

### 1. key 没有统一前缀

后期排查和清理会很痛苦。

### 2. 空值缓存和详情缓存混用

不存在的短码应该使用单独 key。

### 3. 限流 key 不设置 TTL

用户会被永久限制。

### 4. 缓存字段过多

字段越多，更新一致性越难处理。

### 5. 短码只靠内存判断唯一

多实例部署时会出问题。

唯一性必须由数据库约束兜底。

---

## 十、本节练习

请完成下面练习：

1. 写出 `short_links` 表结构。
2. 定义 `ShortLink` 和 `CachedShortLink`。
3. 写出 5 个接口的请求和响应示例。
4. 为 4 类 Redis key 写出示例。
5. 为每类 key 设计 TTL。
6. 思考更新短链接时要删除哪些 key。

---

## 十一、本节小结

这一节完成了项目的基础设计。

你需要记住：

- 数据库保存短链接事实数据。
- Redis key 要有清晰前缀和职责。
- 详情缓存、计数器、限流 key、空值缓存不能混在一起。
- TTL 是缓存设计的一部分，不是附加项。

下一节我们搭建 Go 项目结构并初始化 Redis Client。

