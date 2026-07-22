# 0. Redis 缓存、限流与高级能力总览

第8阶段开始，你会接触 Gin 项目中更接近真实业务的能力：Redis 缓存、限流、文件上传、静态文件服务、超时控制和优雅关闭。

这些内容不是为了“炫技”，而是为了解决后端项目变复杂后必然出现的问题：

- 热点数据每次都查数据库，数据库压力越来越大。
- 登录接口被暴力尝试，账号安全有风险。
- 文件上传没有大小限制，服务可能被大文件拖垮。
- 请求没有超时控制，慢 SQL 或外部服务会拖住整个请求。
- 服务重启时直接杀进程，正在处理的请求被中断。

---

## 一、本阶段学习目标

完成本阶段后，你应该能做到：

- 在 Gin 项目中初始化 Redis 客户端。
- 使用 Redis 缓存用户详情等读多写少的数据。
- 理解缓存穿透、缓存击穿、缓存雪崩的基本处理思路。
- 实现登录失败次数限制。
- 限制文件上传大小、类型和保存路径。
- 使用 Gin 提供静态文件访问。
- 在 handler、service、repository 中传递 `context.Context`。
- 为 HTTP 服务添加启动、超时和优雅关闭逻辑。

---

## 二、本阶段适合解决哪些场景

Redis 和高级能力不是每个小项目一开始都必须用，但你需要知道它们适合解决什么问题。

```text
用户资料、配置、排行榜：适合缓存
登录失败次数：适合 Redis 计数
短信验证码：适合 Redis 设置过期时间
上传头像、附件：需要文件上传控制
慢请求、外部 API：需要 context 超时
服务发布、重启：需要优雅关闭
```

学习时不要把 Redis 当成“更快的数据库”。Redis 更适合做缓存、计数、短期状态和分布式协调。

---

## 三、本阶段目录安排

本阶段文件顺序如下：

```text
0_Redis缓存限流与高级能力总览.md
1_Redis缓存与缓存一致性.md
2_限流文件上传与安全边界.md
3_context优雅关闭与超时控制.md
4_Redis初始化与连接检查.md
5_用户详情缓存实践.md
6_登录失败限流实践.md
7_文件上传与静态文件服务.md
8_本阶段综合实践_缓存限流上传.md
```

建议学习顺序：

1. 先理解缓存和限流的基本概念。
2. 再完成 Redis 初始化。
3. 然后做用户详情缓存。
4. 接着实现登录失败限流。
5. 最后加入文件上传、超时控制和优雅关闭。

---

## 四、本阶段推荐项目结构

在前面项目基础上增加：

```text
gin-user-api/
├── internal/
│   ├── cache/
│   │   └── redis.go
│   ├── middleware/
│   │   └── timeout.go
│   ├── service/
│   └── handler/
├── uploads/
│   └── avatars/
└── config/
    └── config.yaml
```

配置中增加 Redis 和上传配置：

```yaml
redis:
  addr: "127.0.0.1:6379"
  password: ""
  db: 0

upload:
  avatar_dir: "uploads/avatars"
  max_avatar_size_mb: 2
```

---

## 五、学习前置条件

进入本阶段前，你最好已经完成：

- Gin 路由和中间件。
- 统一响应和错误码。
- JWT 登录认证。
- service、repository 分层。
- GORM 用户模块。
- 配置读取。
- 基础测试。

如果还没有完成 Redis 安装，可以使用 Docker 快速启动：

```bash
docker run --name redis-dev -p 6379:6379 -d redis:7
```

验证：

```bash
docker exec -it redis-dev redis-cli ping
```

返回：

```text
PONG
```

说明 Redis 可用。

---

## 六、本阶段最终效果

完成后，你的项目应该支持：

- `GET /api/v1/users/:id` 优先读 Redis 缓存。
- 更新用户资料后删除或刷新缓存。
- 登录失败超过指定次数后短时间内禁止继续登录。
- 上传头像时限制文件大小和扩展名。
- `/static/avatars/...` 可以访问已上传头像。
- 服务关闭时等待正在处理的请求完成。

---

## 七、本阶段验收标准

完成本阶段后，请检查：

- Redis 客户端启动时能 ping 通。
- Redis 不可用时，项目能输出清晰错误。
- 用户详情第一次请求查数据库，第二次请求命中缓存。
- 更新用户资料后不会读到旧缓存。
- 登录失败次数达到上限后返回 `429 Too Many Requests`。
- 上传超过限制大小的文件会被拒绝。
- 上传非图片文件会被拒绝。
- 服务收到退出信号后可以优雅关闭。

第8阶段的重点不是“会用 Redis 命令”，而是知道 Redis 在 Gin 项目中应该放在哪一层、解决什么问题、怎样避免常见坑。

---

## 融合补充

+### 07-01 Redis 缓存、一致性与限流

Redis 是加速和保护层，不是主数据库。先保证没有缓存时业务仍然正确，再针对热点读取增加缓存。

#### Cache-aside

读取用户详情时，先查缓存，未命中再查数据库并设置 TTL；成功更新后删除缓存：

```go
key := fmt.Sprintf("user:%d", id)
value, err := rdb.Get(ctx, key).Result()
if errors.Is(err, redis.Nil) {
    user, err := repo.FindByID(ctx, id)
    if err != nil { return nil, err }
    encoded, _ := json.Marshal(user)
    _ = rdb.Set(ctx, key, encoded, 10*time.Minute).Err()
    return user, nil
}
```

TTL 加少量随机抖动以避免大量键同时失效。缓存穿透可缓存“明确不存在”的短 TTL 结果或使用布隆过滤器；热点击穿使用 singleflight 合并回源；雪崩需要合理 TTL、限流和数据库保护。缓存写失败要记录并降级为数据库读取，不能把缓存当成唯一事实来源。

#### 限流

登录失败限制可按邮箱规范化值或 IP 建键。`INCR` 与设置过期时间需要原子性，生产中使用 Lua、Redis 函数或成熟限流库，避免两步操作造成永不过期键。通用限流可按 IP、用户 ID 或 API key 分桶，超过阈值返回 `429` 和 `Retry-After`。

限流保护的是下游容量，不是认证的替代品；键中不要原样写入无限长度的用户输入。记录命中率、缓存延迟、Redis 错误与被拒绝请求数，才能知道策略是否有效。

**验收：**缓存未命中会回源并写入；更新后旧缓存不再被读取；Redis 不可用时核心接口仍能安全降级；同一凭据连续失败会被短暂限制。

+### 07-02 文件上传、Context、超时与优雅关闭

#### 文件上传边界

先在代理层和应用层限制 body 大小，再验证扩展名、MIME 类型和真实文件内容；不要使用客户端提供的文件路径或名称作为存储路径。

```go
r.MaxMultipartMemory = 8 << 20 // 8 MiB
r.POST("/api/v1/files", func(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil { response.Error(c, 400, 10001, "file is required"); return }
    if file.Size > 5<<20 { response.Error(c, 400, 10001, "file too large"); return }
    name := uuid.NewString() + filepath.Ext(file.Filename)
    if err := c.SaveUploadedFile(file, filepath.Join("uploads", name)); err != nil {
        response.Error(c, 500, 50000, "upload failed"); return
    }
    response.OK(c, gin.H{"name": name})
})
```

上传目录不能执行脚本；下载应考虑鉴权、Content-Disposition 和路径穿越。大文件使用对象存储与预签名上传，避免长期占用 Gin 进程。

#### Context 与超时

把 `c.Request.Context()` 传给 GORM、Redis 和出站 HTTP 调用。每个外部调用设置合理超时；超时不是无限重试的信号。不要将 `*gin.Context` 传入 goroutine 或 Service，异步任务只提取需要的字段并使用自己的 `context.Context`。

#### 优雅关闭

```go
srv := &http.Server{Addr: cfg.Server.Addr, Handler: router}
go func() { if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed { log.Fatal(err) } }()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil { log.Printf("shutdown: %v", err) }
```

关闭时停止接收新请求、等待正在处理的请求、关闭数据库和 Redis。容器的终止宽限期要大于关闭超时，否则会被强制杀死。

**验收：**超大文件被拒绝；客户端取消请求能取消下游调用；收到 SIGTERM 后服务不再接收新请求并在超时内退出。
