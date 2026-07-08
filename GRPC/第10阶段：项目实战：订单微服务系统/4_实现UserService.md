# 4. 实现 UserService

本节目标：实现用户服务，作为订单服务的下游依赖。

UserService 是项目中最简单的服务，适合先完成。它提供用户注册和查询，数据可以先放在内存 map 中，重点练习服务端结构、错误码和测试。

---

## 一、功能设计

- `CreateUser`：创建用户。
- `GetUser`：查询用户。
- 邮箱重复返回 `AlreadyExists`。
- 用户不存在返回 `NotFound`。
- 参数非法返回 `InvalidArgument`。

---

## 二、内存实现示例

```go
type userServer struct {
    userv1.UnimplementedUserServiceServer
    mu     sync.RWMutex
    nextID int64
    users  map[int64]*userv1.User
    emails map[string]int64
}
```

查询用户：

```go
func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id <= 0 {
        return nil, status.Error(codes.InvalidArgument, "id must be positive")
    }

    s.mu.RLock()
    user := s.users[req.Id]
    s.mu.RUnlock()

    if user == nil {
        return nil, status.Error(codes.NotFound, "user not found")
    }

    return &userv1.GetUserResponse{User: user}, nil
}
```

---

## 三、启动服务

```go
lis, err := net.Listen("tcp", ":50051")
if err != nil {
    log.Fatal(err)
}

s := grpc.NewServer()
userv1.RegisterUserServiceServer(s, newUserServer())
reflection.Register(s)

log.Fatal(s.Serve(lis))
```

---

## 四、常见问题

- map 并发读写不加锁：压测或并发调用时会 panic。
- 不区分重复邮箱和内部错误：调用方无法处理。
- 返回内部指针后被外部修改：真实项目要注意复制或不可变对象。
- 没有测试错误码：调用方依赖的契约没有保障。

---

## 五、练习任务

1. 实现 CreateUser 和 GetUser。
2. 写 4 个测试：成功、非法参数、不存在、重复邮箱。
3. 用 grpcurl 调用 GetUser。
4. 给服务加 health check。

---

## 六、完成标准

- UserService 可独立启动。
- 错误码符合规范。
- 测试覆盖主要路径。

---

## 七、完整操作步骤：先完成最小下游服务

UserService 是 order-service 的下游依赖，因此优先实现它。建议顺序如下：

1. 确认 `user.proto` 已经生成 Go 代码。
2. 在 `internal/app/user` 中实现 server struct。
3. 使用内存 map 保存用户。
4. 实现 `CreateUser` 和 `GetUser`。
5. 在 `cmd/user-service/main.go` 注册服务。
6. 用 grpcurl 调用成功路径和错误路径。
7. 补充单元测试或 bufconn 测试。

创建目录：

```powershell
mkdir .\internal\app\user -Force
New-Item -ItemType File .\internal\app\user\service.go -Force
```

---

## 八、完整代码：UserService 内存实现

`internal/app/user/service.go`：

```go
package user

import (
	"context"
	"strings"
	"sync"

	userv1 "example.com/grpc-shop/gen/user/v1"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

type Service struct {
	userv1.UnimplementedUserServiceServer

	mu     sync.RWMutex
	nextID int64
	users  map[int64]*userv1.User
	emails map[string]int64
}

func NewService() *Service {
	s := &Service{
		nextID: 3,
		users: map[int64]*userv1.User{
			1: {Id: 1, Name: "alice", Email: "alice@example.com"},
			2: {Id: 2, Name: "bob", Email: "bob@example.com"},
		},
		emails: map[string]int64{
			"alice@example.com": 1,
			"bob@example.com":   2,
		},
	}
	return s
}

func (s *Service) CreateUser(ctx context.Context, req *userv1.CreateUserRequest) (*userv1.CreateUserResponse, error) {
	name := strings.TrimSpace(req.GetName())
	email := strings.TrimSpace(req.GetEmail())
	if name == "" || email == "" {
		return nil, status.Error(codes.InvalidArgument, "name and email are required")
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	if _, ok := s.emails[email]; ok {
		return nil, status.Error(codes.AlreadyExists, "email already exists")
	}

	id := s.nextID
	s.nextID++

	user := &userv1.User{Id: id, Name: name, Email: email}
	s.users[id] = user
	s.emails[email] = id

	return &userv1.CreateUserResponse{User: cloneUser(user)}, nil
}

func (s *Service) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	if req.GetId() <= 0 {
		return nil, status.Error(codes.InvalidArgument, "id must be positive")
	}

	s.mu.RLock()
	defer s.mu.RUnlock()

	user, ok := s.users[req.GetId()]
	if !ok {
		return nil, status.Error(codes.NotFound, "user not found")
	}

	return &userv1.GetUserResponse{User: cloneUser(user)}, nil
}

func cloneUser(u *userv1.User) *userv1.User {
	if u == nil {
		return nil
	}
	return &userv1.User{Id: u.Id, Name: u.Name, Email: u.Email}
}
```

注册服务：

```go
grpcServer := server.NewGRPCServer()
userv1.RegisterUserServiceServer(grpcServer, user.NewService())
```

---

## 九、运行命令与预期输出

启动服务：

```powershell
go run .\cmd\user-service
```

预期输出：

```text
user-service listening on :50051
```

调用成功路径：

```powershell
grpcurl -plaintext -d '{\"id\":1}' localhost:50051 user.v1.UserService/GetUser
```

预期输出：

```json
{
  "user": {
    "id": "1",
    "name": "alice",
    "email": "alice@example.com"
  }
}
```

调用错误路径：

```powershell
grpcurl -plaintext -d '{\"id\":999}' localhost:50051 user.v1.UserService/GetUser
```

预期输出包含：

```text
Code: NotFound
Message: user not found
```

---

## 十、常见错误排查

- `unknown service user.v1.UserService`：服务没有注册，检查 `RegisterUserServiceServer`。
- `Unimplemented`：proto 生成代码更新了，但服务实现没有实现对应方法。
- 重复邮箱没有报错：检查 `emails` 索引是否在创建用户时同步更新。
- 并发测试偶发失败：map 读写必须加锁。
- 返回用户后被外部修改：内存对象建议 clone 后返回，避免共享指针带来隐蔽问题。

---

## 十一、教程闭环检查

本篇的完整操作步骤是：创建 user app 目录、实现服务、注册服务、启动服务、用 grpcurl 验证成功和失败路径。完整代码包含 UserService 的内存实现、并发锁、错误码和 clone。运行命令包含 `go run` 和两条 `grpcurl`。预期输出覆盖用户查询成功和 NotFound。常见错误排查覆盖服务注册、Unimplemented、重复邮箱、并发 map 和共享指针。练习任务是补充 CreateUser 的 grpcurl 调用和四个测试用例。完成标准是：UserService 可以作为稳定下游被 OrderService 调用。
