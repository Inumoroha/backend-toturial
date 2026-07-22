# 5. Service 测试与表格驱动

本节目标：为 service 层编写表格驱动测试，重点验证业务规则，而不是 HTTP 细节。

handler 测试关注“请求进来以后返回什么 HTTP 响应”，service 测试关注“业务规则是否正确”。这两类测试要分开，否则项目越大，测试越难维护。

---

## 一、service 层应该测试什么

以用户注册为例，service 层可能包含这些规则：

- 邮箱不能为空。
- 密码长度不能小于 6 位。
- 邮箱不能重复。
- 密码入库前必须加密。
- repository 创建失败时要返回错误。
- 创建成功后返回用户 DTO，而不是数据库模型。

这些规则不依赖 HTTP，所以不应该用 `httptest` 测。直接调用 service 方法更清晰。

---

## 二、准备 service 接口

示例：

```go
type UserRepository interface {
    FindByEmail(ctx context.Context, email string) (*model.User, error)
    Create(ctx context.Context, user *model.User) error
}

type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

type RegisterRequest struct {
    Email    string
    Password string
    Nickname string
}

type UserDTO struct {
    ID    uint
    Email string
}
```

service 依赖 repository 接口，这样测试时可以传入 fake repository。

---

## 三、编写 fake repository

```go
type fakeUserRepository struct {
    findByEmailFunc func(ctx context.Context, email string) (*model.User, error)
    createFunc      func(ctx context.Context, user *model.User) error
}

func (f *fakeUserRepository) FindByEmail(ctx context.Context, email string) (*model.User, error) {
    if f.findByEmailFunc != nil {
        return f.findByEmailFunc(ctx, email)
    }
    return nil, gorm.ErrRecordNotFound
}

func (f *fakeUserRepository) Create(ctx context.Context, user *model.User) error {
    if f.createFunc != nil {
        return f.createFunc(ctx, user)
    }
    user.ID = 1
    return nil
}
```

fake repository 的作用是控制 repository 的返回值，从而测试 service 在不同场景下的表现。

---

## 四、什么是表格驱动测试

表格驱动测试就是把多个测试场景写成一个切片：

```go
tests := []struct {
    name    string
    req     RegisterRequest
    repo    *fakeUserRepository
    wantErr bool
}{
    {
        name: "success",
        req: RegisterRequest{
            Email:    "a@example.com",
            Password: "123456",
        },
        repo:    &fakeUserRepository{},
        wantErr: false,
    },
    {
        name: "email exists",
        req: RegisterRequest{
            Email:    "a@example.com",
            Password: "123456",
        },
        repo: &fakeUserRepository{
            findByEmailFunc: func(ctx context.Context, email string) (*model.User, error) {
                return &model.User{ID: 1, Email: email}, nil
            },
        },
        wantErr: true,
    },
}
```

再用循环执行：

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        svc := NewUserService(tt.repo)
        got, err := svc.Register(context.Background(), tt.req)

        if tt.wantErr && err == nil {
            t.Fatal("want error, got nil")
        }

        if !tt.wantErr && err != nil {
            t.Fatalf("want nil error, got %v", err)
        }

        if !tt.wantErr && got.Email != tt.req.Email {
            t.Fatalf("want email %s, got %s", tt.req.Email, got.Email)
        }
    })
}
```

优点：

- 用例结构清晰。
- 新增边界条件很方便。
- 测试名称会显示在 `go test -v` 输出里。

---

## 五、完整注册测试示例

```go
func TestUserService_Register(t *testing.T) {
    tests := []struct {
        name    string
        req     RegisterRequest
        repo    *fakeUserRepository
        wantErr bool
    }{
        {
            name: "success",
            req: RegisterRequest{
                Email:    "a@example.com",
                Password: "123456",
                Nickname: "alice",
            },
            repo:    &fakeUserRepository{},
            wantErr: false,
        },
        {
            name: "email already exists",
            req: RegisterRequest{
                Email:    "a@example.com",
                Password: "123456",
            },
            repo: &fakeUserRepository{
                findByEmailFunc: func(ctx context.Context, email string) (*model.User, error) {
                    return &model.User{ID: 1, Email: email}, nil
                },
            },
            wantErr: true,
        },
        {
            name: "repository create failed",
            req: RegisterRequest{
                Email:    "a@example.com",
                Password: "123456",
            },
            repo: &fakeUserRepository{
                createFunc: func(ctx context.Context, user *model.User) error {
                    return errors.New("database error")
                },
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := NewUserService(tt.repo)
            got, err := svc.Register(context.Background(), tt.req)

            if tt.wantErr {
                if err == nil {
                    t.Fatal("want error, got nil")
                }
                return
            }

            if err != nil {
                t.Fatalf("want nil error, got %v", err)
            }

            if got == nil {
                t.Fatal("want user, got nil")
            }

            if got.Email != tt.req.Email {
                t.Fatalf("want email %s, got %s", tt.req.Email, got.Email)
            }
        })
    }
}
```

---

## 六、测试错误类型

如果项目中定义了业务错误：

```go
var ErrEmailExists = errors.New("email already exists")
```

测试时不要只判断 `err != nil`，可以进一步判断：

```go
if !errors.Is(err, ErrEmailExists) {
    t.Fatalf("want ErrEmailExists, got %v", err)
}
```

这样 handler 层才能根据错误类型转换成正确的响应：

```go
if errors.Is(err, service.ErrEmailExists) {
    response.Fail(c, http.StatusConflict, "EMAIL_EXISTS", "邮箱已存在")
    return
}
```

---

## 七、测试密码加密

注册成功时，还应该确认入库前密码不是明文：

```go
createFunc: func(ctx context.Context, user *model.User) error {
    if user.PasswordHash == "123456" {
        t.Fatal("password should not be stored as plain text")
    }

    if user.PasswordHash == "" {
        t.Fatal("password hash should not be empty")
    }

    user.ID = 1
    return nil
}
```

这类测试很有价值，因为它能防止以后重构时不小心把密码明文写入数据库。

---

## 八、常见问题

### 1. service 测试写成了 HTTP 测试

如果你在 service 测试中使用了 `httptest`，说明层次混了。service 测试应该直接调用方法。

### 2. 每个测试都要准备很多 fake 方法

可以为 fake repository 提供默认行为。只有当前测试关心的行为才覆盖。

### 3. 只测成功，不测失败

后端项目里失败分支非常重要。参数错误、重复数据、数据库错误、权限错误都应该有测试。

---

## 九、练习

请为你的项目补充 service 测试：

1. 注册成功。
2. 邮箱已存在。
3. repository 创建失败。
4. 密码不能明文保存。
5. 登录成功。
6. 登录时用户不存在。
7. 登录时密码错误。

---

## 十、验收标准

运行：

```bash
go test ./internal/service -v
```

确认：

- service 测试不依赖 Gin。
- service 测试不启动 HTTP 服务。
- 关键业务规则都有成功和失败用例。
- 表格驱动测试中每个 case 名称清楚。
- 错误类型能被准确断言。

完成后，你就可以进入 Swagger 接口文档部分。

