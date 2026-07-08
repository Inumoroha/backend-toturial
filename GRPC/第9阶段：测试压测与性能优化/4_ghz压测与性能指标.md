# 4. ghz 压测与性能指标

本节目标：使用 ghz 对 gRPC 服务做基础压测，并读懂关键指标。

压测不是为了追求漂亮数字，而是为了知道服务在一定并发下的延迟、吞吐、错误率和瓶颈。gRPC 可以使用 ghz 做命令行压测。

---

## 一、核心指标

- QPS：每秒请求数，代表吞吐。
- Average latency：平均延迟，只能作为参考。
- P95/P99：95%/99% 请求的延迟，通常更重要。
- Error rate：错误率，高 QPS 但大量失败没有意义。
- CPU、内存、goroutine、GC：定位瓶颈时必须结合系统指标。

---

## 二、基本命令

```bash
ghz --insecure \
  --proto proto/user/v1/user.proto \
  --call user.v1.UserService.GetUser \
  -d '{"id":1}' \
  -c 20 -n 1000 \
  localhost:50051
```

参数说明：

- `--proto`：指定 proto 文件。
- `--call`：指定完整 RPC 方法。
- `-d`：请求数据。
- `-c`：并发数。
- `-n`：总请求数。

---

## 三、压测记录模板

```text
服务：UserService.GetUser
机器：本地 Windows
并发：20
请求数：1000
QPS：xxx
平均延迟：xxx ms
P95：xxx ms
P99：xxx ms
错误率：x%
结论：...
```

每次压测都要记录参数，否则结果无法复现。

---

## 四、常见问题

- 在自己电脑上压测后直接推断生产性能：本地结果只能做参考。
- 忽略错误率：高 QPS 但大量失败没有意义。
- 压测没有预热：第一次结果可能受冷启动影响。
- 只改并发不改请求体：无法观察不同 payload 对性能的影响。

---

## 五、练习任务

1. 用并发 1、10、50 分别压测。
2. 记录 QPS、P95、P99。
3. 改变服务端 sleep 时间，观察结果变化。
4. 用 pprof 或运行时指标观察 goroutine 数量。

---

## 六、完成标准

- 能运行一次 ghz 压测。
- 能读懂核心指标。
- 能根据结果提出初步优化方向。

---

## 七、完整操作步骤：从安装到第一次压测

本节建议你不要一上来就追求高并发，而是先跑通一条稳定、可重复的压测链路。压测链路包括四件事：服务可以启动、proto 路径正确、请求数据合法、结果可以记录。

### 1. 安装 ghz

如果你已经安装了 Go，可以直接使用下面的命令安装：

```powershell
go install github.com/bojand/ghz/cmd/ghz@latest
```

安装后检查命令是否可用：

```powershell
ghz --version
```

预期输出类似：

```text
ghz dev
```

如果 PowerShell 提示找不到 `ghz`，通常是 Go 的 bin 目录没有加入 `PATH`。先查看 Go bin 目录：

```powershell
go env GOPATH
```

假设输出是：

```text
C:\Users\你的用户名\go
```

那么 `ghz.exe` 通常位于：

```text
C:\Users\你的用户名\go\bin\ghz.exe
```

你可以临时加入当前 PowerShell 会话：

```powershell
$env:Path += ';' + (Join-Path (go env GOPATH) 'bin')
ghz --version
```

如果你不想通过 `go install` 安装，也可以下载 ghz 的 release 二进制文件。学习阶段推荐使用 `go install`，因为路径和版本更容易统一记录。

### 2. 启动被压测的 gRPC 服务

压测前先启动服务端。以下命令假设你已经按照前面阶段准备好了示例项目：

```powershell
go run .\cmd\server
```

服务端预期输出：

```text
grpc server listening on :50051
```

如果你的项目入口不同，以项目实际入口为准，例如：

```powershell
go run .\cmd\user-server
```

压测时不要把服务端和 ghz 的输出混在同一个终端里。建议打开两个 PowerShell 窗口：

- 第一个窗口启动服务端。
- 第二个窗口执行 ghz 压测命令。

### 3. 先用 grpcurl 验证接口可用

压测前先做一次普通调用，避免把“接口本来就不可用”误判成“性能不好”：

```powershell
grpcurl -plaintext -d '{\"id\":1}' localhost:50051 user.v1.UserService/GetUser
```

预期输出类似：

```json
{
  "user": {
    "id": "1",
    "name": "alice",
    "email": "alice@example.com"
  }
}
```

如果这一步失败，先修服务，不要急着压测。压测只会放大问题，不会帮你绕过问题。

### 4. 执行第一次 ghz 压测

PowerShell 对 JSON 引号比较敏感，推荐先使用单行命令，并用单引号包裹 JSON：

```powershell
ghz --insecure --proto .\proto\user\v1\user.proto --call user.v1.UserService.GetUser -d '{\"id\":1}' -c 20 -n 1000 localhost:50051
```

如果你的 proto 文件依赖公共目录，可以补充 `--import-path`：

```powershell
ghz --insecure `
  --import-path . `
  --import-path .\proto `
  --proto .\proto\user\v1\user.proto `
  --call user.v1.UserService.GetUser `
  -d '{\"id\":1}' `
  -c 20 `
  -n 1000 `
  localhost:50051
```

这里使用反引号换行，是 PowerShell 的换行方式。反引号后面不要有空格，否则命令可能被截断。

---

## 八、完整代码：准备一个可压测的 GetUser 服务

如果你没有现成服务，可以使用下面这个极简服务作为压测目标。它故意使用内存数据，避免数据库波动影响你第一次理解 ghz 指标。

```go
package main

import (
	"context"
	"log"
	"net"

	userv1 "example.com/grpc-demo/gen/user/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

type userServer struct {
	userv1.UnimplementedUserServiceServer
	users map[int64]*userv1.User
}

func newUserServer() *userServer {
	return &userServer{
		users: map[int64]*userv1.User{
			1: {Id: 1, Name: "alice", Email: "alice@example.com"},
			2: {Id: 2, Name: "bob", Email: "bob@example.com"},
		},
	}
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	if req.GetId() <= 0 {
		return nil, status.Error(codes.InvalidArgument, "id must be positive")
	}

	user, ok := s.users[req.GetId()]
	if !ok {
		return nil, status.Error(codes.NotFound, "user not found")
	}

	return &userv1.GetUserResponse{User: user}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}

	server := grpc.NewServer()
	userv1.RegisterUserServiceServer(server, newUserServer())

	log.Println("grpc server listening on :50051")
	if err := server.Serve(lis); err != nil {
		log.Fatalf("serve: %v", err)
	}
}
```

这段代码适合做基础压测，但不适合代表真实生产性能。真实服务还会包含数据库、缓存、日志、鉴权、链路追踪、网络抖动等因素。

---

## 九、运行命令：分层压测，而不是只跑一次

建议至少跑三组并发，每组先预热，再正式记录。

### 1. 预热

```powershell
ghz --insecure --proto .\proto\user\v1\user.proto --call user.v1.UserService.GetUser -d '{\"id\":1}' -c 5 -n 100 localhost:50051
```

预热结果不写进最终报告，只用于让服务进入相对稳定状态。

### 2. 低并发基线

```powershell
ghz --insecure --proto .\proto\user\v1\user.proto --call user.v1.UserService.GetUser -d '{\"id\":1}' -c 1 -n 200 localhost:50051
```

这一组用来观察单请求链路的基础延迟。如果低并发下 P95 就很高，通常说明服务内部逻辑、日志、依赖调用或本机环境已经存在明显问题。

### 3. 中等并发

```powershell
ghz --insecure --proto .\proto\user\v1\user.proto --call user.v1.UserService.GetUser -d '{\"id\":1}' -c 20 -n 1000 localhost:50051
```

这一组可以作为日常学习阶段的主要观察对象。它能让你看到吞吐、平均延迟和尾延迟之间的关系。

### 4. 高并发探索

```powershell
ghz --insecure --proto .\proto\user\v1\user.proto --call user.v1.UserService.GetUser -d '{\"id\":1}' -c 50 -n 3000 localhost:50051
```

高并发不是越高越好。并发数升高后，如果 QPS 不再增长，P95/P99 快速上升，错误率增加，就说明服务或本机资源已经接近瓶颈。

---

## 十、预期输出：如何读懂 ghz 报告

ghz 输出通常包含以下信息：

```text
Summary:
  Count:        1000
  Total:        1.24 s
  Slowest:      45.12 ms
  Fastest:      1.20 ms
  Average:      8.32 ms
  Requests/sec: 806.45

Response time histogram:
  1.200 [1]    |
  5.592 [521]  |■■■■■■■■■■■■■■■■■■
  9.984 [351]  |■■■■■■■■■■■■
  14.376 [100] |■■■■

Latency distribution:
  10 % in 2.11 ms
  25 % in 3.70 ms
  50 % in 6.42 ms
  75 % in 10.35 ms
  90 % in 18.40 ms
  95 % in 25.90 ms
  99 % in 40.10 ms

Status code distribution:
  [OK]   1000 responses
```

阅读顺序建议如下：

1. 先看 `Status code distribution`，确认是不是大多数请求都成功。
2. 再看 `Requests/sec`，了解吞吐。
3. 再看 `Average`，了解大致平均水平。
4. 重点看 `95 %` 和 `99 %`，判断尾延迟是否可接受。
5. 最后看直方图，观察是否出现明显长尾。

平均延迟很容易骗人。比如 990 个请求都是 3ms，但 10 个请求是 1000ms，平均值可能看起来还行，用户体验却已经很差。因此服务端性能报告一定要记录 P95/P99。

---

## 十一、常见错误排查

### 1. `unknown service user.v1.UserService`

通常是 `--call` 写错了。ghz 使用的是 proto 里的完整服务名和方法名：

```text
包名.服务名.方法名
```

例如：

```powershell
--call user.v1.UserService.GetUser
```

而 grpcurl 常见写法是：

```powershell
user.v1.UserService/GetUser
```

两者分隔符不同，不要混用。

### 2. `open proto/user/v1/user.proto: The system cannot find the path specified`

说明当前目录不对，或者 `--proto` 路径写错。先在项目根目录执行：

```powershell
Get-ChildItem .\proto\user\v1\user.proto
```

如果找不到文件，回到项目根目录再执行 ghz。

### 3. `import was not found or had errors`

说明 proto 里引用了其他文件，但 ghz 没有找到 import path。补充：

```powershell
--import-path . --import-path .\proto
```

如果使用了 `google/api/annotations.proto`，还需要确认第三方 proto 文件也在 import path 中。

### 4. `connection refused`

说明服务没有启动，或者端口不是 `50051`。检查服务端日志，并确认监听地址：

```powershell
netstat -ano | Select-String ':50051'
```

### 5. 错误率突然升高

先不要急着优化代码，按顺序检查：

- 请求数据是不是触发了业务错误。
- 服务端是否出现 panic 或 deadline exceeded。
- 并发数是否超过本机承载能力。
- 是否打印了过多同步日志。
- 是否有数据库、缓存、外部 HTTP 调用等慢依赖。

---

## 十二、压测报告模板

每次压测都建议留下报告，哪怕只是学习项目。报告不是写给别人看的形式主义，而是帮助你复盘实验条件。

```text
测试对象：user.v1.UserService.GetUser
测试时间：2026-07-05 20:00
机器环境：Windows，本地开发机
服务版本：本地 main 分支
ghz 版本：通过 ghz --version 记录
请求数据：{"id":1}
总请求数：1000
并发数：20
是否预热：是，100 次

结果：
- Requests/sec：
- Average：
- P50：
- P95：
- P99：
- Error rate：
- Status code distribution：

观察：
- QPS 是否随并发增加而增加：
- P95/P99 是否出现明显长尾：
- 服务端 CPU/内存/goroutine 是否异常：

结论：
- 当前瓶颈猜测：
- 下一步验证计划：
```

---

## 十三、教程闭环检查

本篇教程的颗粒度需要达到下面的标准：

- 有完整操作步骤：安装 ghz、启动服务、验证接口、执行压测、记录报告。
- 有完整代码：提供了可被压测的 `GetUser` 服务端示例。
- 有运行命令：包含 `go install`、`go run`、`grpcurl`、`ghz`、`netstat` 等命令。
- 有预期输出：包含服务启动输出、grpcurl 输出、ghz 结果结构。
- 有常见错误排查：覆盖 proto 路径、import path、方法名、连接失败和错误率升高。
- 有练习任务：要求对比并发 1、20、50 的 QPS、P95、P99。
- 有完成标准：能够独立完成一次可复现的 gRPC 压测，并能解释报告指标。

完成这一篇后，你不需要成为性能专家，但应该能做到：看到一份 ghz 报告时，不只盯着 QPS，而是能同时判断成功率、平均延迟、尾延迟和实验条件是否可信。
