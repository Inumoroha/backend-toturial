# 08-01 Redis 缓存、限流、文件上传与优雅关闭

本阶段目标：学习真实业务中常见的高级能力，包括缓存、限流、文件上传、请求超时和服务优雅关闭。

## 一、Redis 能做什么

Redis 常见用途：

- 缓存热点数据。
- 保存登录态或 token 黑名单。
- 接口限流。
- 分布式锁。
- 排行榜和计数器。

本阶段重点学习缓存和限流。

## 二、安装 Redis 客户端

```bash
go get github.com/redis/go-redis/v9
```

初始化：

```go
func NewRedis() *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr:     "127.0.0.1:6379",
        Password: "",
        DB:       0,
    })
}
```

注意 Redis v9 使用 `context.Context`：

```go
ctx := context.Background()
value, err := rdb.Get(ctx, "key").Result()
```

## 三、缓存用户详情

缓存 key 设计：

```text
user:detail:1
```

查询流程：

1. 先查 Redis。
2. Redis 命中，反序列化后直接返回。
3. Redis 未命中，查数据库。
4. 数据库查到后写入 Redis。
5. 返回结果。

示例：

```go
func (s *UserService) GetUser(ctx context.Context, id uint) (*model.User, error) {
    key := fmt.Sprintf("user:detail:%d", id)

    cached, err := s.redis.Get(ctx, key).Result()
    if err == nil {
        var user model.User
        if json.Unmarshal([]byte(cached), &user) == nil {
            return &user, nil
        }
    }

    user, err := s.repo.GetByID(id)
    if err != nil {
        return nil, err
    }

    bytes, _ := json.Marshal(user)
    s.redis.Set(ctx, key, bytes, 10*time.Minute)

    return user, nil
}
```

更新用户时要删除缓存：

```go
s.redis.Del(ctx, fmt.Sprintf("user:detail:%d", user.ID))
```

这叫缓存失效。初学阶段优先使用“更新数据库后删除缓存”的方式。

## 四、缓存常见问题

### 1. 缓存穿透

查询一个数据库里根本不存在的数据，每次都打到数据库。

解决思路：

- 对空结果短暂缓存。
- 使用布隆过滤器。
- 做参数校验，拦截明显非法 ID。

### 2. 缓存击穿

一个热点 key 过期，大量请求同时打到数据库。

解决思路：

- 热点数据设置更长过期时间。
- 加互斥锁。
- 后台异步刷新缓存。

### 3. 缓存雪崩

大量 key 同时过期，数据库压力暴增。

解决思路：

- 过期时间加随机值。
- 分批预热缓存。
- 限流和降级。

## 五、登录失败限流

例如同一邮箱 5 分钟内最多失败 5 次：

```go
func (s *AuthService) CheckLoginLimit(ctx context.Context, email string) error {
    key := "login:fail:" + email
    count, err := s.redis.Get(ctx, key).Int()
    if err == redis.Nil {
        return nil
    }
    if err != nil {
        return err
    }
    if count >= 5 {
        return ErrTooManyLoginAttempts
    }
    return nil
}

func (s *AuthService) RecordLoginFail(ctx context.Context, email string) {
    key := "login:fail:" + email
    count, _ := s.redis.Incr(ctx, key).Result()
    if count == 1 {
        s.redis.Expire(ctx, key, 5*time.Minute)
    }
}
```

登录成功后可以删除失败记录：

```go
s.redis.Del(ctx, "login:fail:"+email)
```

## 六、文件上传

头像上传接口：

```go
func UploadAvatar(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        response.Fail(c, http.StatusBadRequest, 40001, "file required")
        return
    }

    if file.Size > 2*1024*1024 {
        response.Fail(c, http.StatusBadRequest, 40001, "file too large")
        return
    }

    filename := uuid.NewString() + path.Ext(file.Filename)
    dst := path.Join("uploads", filename)

    if err := c.SaveUploadedFile(file, dst); err != nil {
        response.Fail(c, http.StatusInternalServerError, 50001, "upload failed")
        return
    }

    response.OK(c, gin.H{
        "url": "/uploads/" + filename,
    })
}
```

静态文件访问：

```go
r.Static("/uploads", "./uploads")
```

注意：

- 生产环境最好上传到对象存储。
- 要限制文件大小。
- 要校验文件类型。
- 不要直接相信用户上传的文件名。

## 七、请求超时控制

handler 里可以使用请求自带的 context：

```go
ctx := c.Request.Context()
```

调用数据库、Redis、外部接口时传递它：

```go
user, err := service.GetUser(ctx, id)
```

这样请求取消或超时时，后续操作也能尽快结束。

## 八、优雅关闭

不要直接：

```go
r.Run(":8080")
```

生产环境建议使用 `http.Server`：

```go
srv := &http.Server{
    Addr:    ":8080",
    Handler: r,
}

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Fatal("server forced to shutdown:", err)
}
```

优雅关闭会给正在处理的请求一点时间完成。

## 九、阶段练习

完成：

- 使用 Redis 缓存用户详情。
- 更新用户时删除缓存。
- 登录失败次数限制。
- 上传头像接口。
- 静态文件访问。
- 服务优雅关闭。
- 在 service/repository 中传递 `context.Context`。

## 十、验收清单

你应该能够回答：

- 什么是缓存穿透、击穿、雪崩？
- 为什么更新数据库后要处理缓存？
- Redis 限流的基本思路是什么？
- 文件上传为什么要限制大小和类型？
- 什么是优雅关闭？

