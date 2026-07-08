# 2. 密码认证、ACL 与最小权限

Redis 安全的第一层是认证。

但生产中只设置一个全局密码还不够。更好的方式是使用 ACL，为不同应用分配不同用户、不同 key 前缀和不同命令权限。

学完这一节后，你应该能够：

- 配置 Redis 密码认证。
- 理解 Redis ACL 的基本概念。
- 创建应用专用用户。
- 限制用户只能访问指定 key 前缀。
- 按最小权限设计 Redis 用户。

---

## 一、基础密码认证

配置：

```conf
requirepass strong-password
```

客户端连接后认证：

```redis
AUTH strong-password
```

使用 redis-cli：

```bash
redis-cli -a strong-password
```

或者：

```bash
redis-cli
AUTH strong-password
```

---

## 二、Go 中配置密码

```go
rdb := redis.NewClient(&redis.Options{
    Addr:     "127.0.0.1:6379",
    Password: "strong-password",
    DB:       0,
})
```

如果认证失败，命令会返回类似：

```text
NOAUTH Authentication required.
WRONGPASS invalid username-password pair
```

不要把密码硬编码在代码里。

应该通过环境变量或密钥管理系统注入。

---

## 三、为什么只用 requirepass 不够

如果所有服务共用一个密码：

```text
article-service
order-service
admin-service
```

任何一个服务泄露密码，都可能访问所有 key，执行所有命令。

风险：

- 删除别的业务 key。
- 执行危险命令。
- 误扫全库。
- 泄露敏感数据。

ACL 可以降低影响范围。

---

## 四、ACL 是什么

ACL 是 Access Control List。

它可以控制：

- 用户是否启用。
- 用户密码。
- 可以访问哪些 key。
- 可以执行哪些命令。
- 可以订阅哪些 channel。

查看用户：

```redis
ACL LIST
```

查看当前用户：

```redis
ACL WHOAMI
```

---

## 五、创建应用用户

创建一个缓存读写用户：

```redis
ACL SETUSER article_app on >article-password ~article:* +get +set +del +expire +ttl
```

含义：

- `article_app`：用户名。
- `on`：启用用户。
- `>article-password`：设置密码。
- `~article:*`：只能访问 `article:*` key。
- `+get +set +del +expire +ttl`：允许这些命令。

这样它不能访问 `order:*`。

也不能执行 `FLUSHALL`。

---

## 六、Go 使用 ACL 用户

```go
rdb := redis.NewClient(&redis.Options{
    Addr:     "127.0.0.1:6379",
    Username: "article_app",
    Password: "article-password",
    DB:       0,
})
```

Redis 6 以后支持用户名密码认证。

如果使用旧版本，要确认客户端和 Redis 版本能力。

---

## 七、只读用户

创建只读用户：

```redis
ACL SETUSER report_reader on >reader-password ~article:* +get +mget +exists +ttl
```

适合：

- 报表查询。
- 只读后台。
- 观测工具。

只读用户不应该有：

- `+set`
- `+del`
- `+flushall`
- `+config`

---

## 八、命令分类授权

Redis ACL 支持命令分类。

例如：

```redis
ACL SETUSER cache_user on >pwd ~cache:* +@read +@write -flushall -flushdb -keys -config
```

含义：

- 允许读写类命令。
- 禁止 `flushall`、`flushdb`、`keys`、`config`。

分类很方便，但生产中仍建议明确排除危险命令。

---

## 九、禁用默认用户

Redis 默认用户是：

```text
default
```

生产中可以考虑禁用默认用户：

```redis
ACL SETUSER default off
```

但要先确保应用已经切换到新用户。

否则会导致服务无法连接。

操作前要有回滚方案。

---

## 十、ACL 配置持久化

ACL 可以写在配置文件中，也可以使用 ACL 文件。

示例：

```conf
aclfile /etc/redis/users.acl
```

运行时修改后：

```redis
ACL SAVE
```

生产中要把 ACL 配置纳入版本管理或配置管理系统。

不要只靠手工命令。

---

## 十一、密码管理

建议：

- 不把密码提交到 Git。
- 使用环境变量、密钥系统或配置中心。
- 定期轮换。
- 不同服务不同账号。
- 离职或泄露后能单独吊销。
- 日志中不要打印密码。

示例环境变量：

```text
REDIS_USERNAME=article_app
REDIS_PASSWORD=...
```

---

## 十二、常见错误

### 1. 没有密码

一旦网络暴露，风险极高。

### 2. 所有服务共用 default 用户

权限过大，无法隔离。

### 3. ACL key 前缀太宽

例如 `~*` 等于能访问所有 key。

### 4. 给业务用户 CONFIG 权限

业务应用不应该修改 Redis 配置。

### 5. 密码写进代码仓库

泄露后很难控制影响。

---

## 十三、本节练习

请完成下面练习：

1. 给 Redis 设置 `requirepass`。
2. 使用 `AUTH` 认证。
3. 创建 `article_app` ACL 用户。
4. 限制它只能访问 `article:*`。
5. 在 Go 中使用用户名和密码连接。
6. 创建一个只读用户。
7. 尝试用只读用户执行 `SET`，观察错误。
8. 思考如何轮换 Redis 密码。

---

## 十四、本节小结

这一节你学习了 Redis 认证和 ACL。

你需要记住：

- `requirepass` 是基础认证。
- Redis ACL 可以控制用户、key 前缀和命令权限。
- 生产中推荐不同服务使用不同 Redis 用户。
- 权限要最小化，不要给业务用户危险命令。
- 密码要通过安全方式管理，不能硬编码。

下一节我们学习网络暴露、绑定地址和 TLS 基础。

