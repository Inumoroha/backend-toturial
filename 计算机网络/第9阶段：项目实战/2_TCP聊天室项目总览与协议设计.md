# 2. TCP 聊天室项目总览与协议设计

本节目标：设计 TCP 聊天室项目的功能、目录和文本协议。

---

## 一、项目目标

实现一个命令行聊天室：

- 多客户端连接。
- 设置昵称。
- 广播消息。
- 查看在线用户。
- 退出清理连接。
- 后续增加心跳和私聊。

---

## 二、目录结构

```text
tcp-chatroom/
  go.mod
  cmd/server/main.go
  internal/chat/server.go
  internal/chat/client.go
```

创建：

```bash
mkdir tcp-chatroom
cd tcp-chatroom
go mod init tcp-chatroom
mkdir -p cmd/server internal/chat
```

---

## 三、协议设计

使用文本行协议：

```text
/name alice
/users
/quit
hello everyone
```

响应：

```text
system: welcome
alice: hello
error: nickname exists
```

为什么用换行？

因为 TCP 是字节流，必须定义消息边界。这里用 `\n` 作为边界。

---

## 四、核心结构

```go
type Client struct {
    Name string
    Conn net.Conn
    Send chan string
}

type Server struct {
    mu      sync.Mutex
    clients map[*Client]struct{}
    names   map[string]*Client
}
```

`Send` channel 用于避免多个 goroutine 同时写同一个连接。

---

## 补充落地步骤：第一天先完成什么

不要一开始就做完整聊天室。第一天只完成三件事：

```text
1. 服务端监听端口。
2. 客户端能连接并发送一行文本。
3. 服务端能把这行文本原样 echo 回去。
```

完成后再加：

```text
/name 设置昵称。
/users 查看在线用户。
普通文本广播。
/quit 退出。
```

每加一个命令，都先写清楚：

```text
客户端输入格式。
服务端如何解析。
成功响应是什么。
失败响应是什么。
连接是否继续保持。
```

这样项目会稳步推进，而不是一口气把连接管理、广播、心跳全揉在一起。

---

## 补充验收：协议设计是否合格

你的协议设计至少要回答：

```text
消息边界是什么？这里是换行。
昵称最大长度是多少？
昵称重复怎么办？
空消息怎么办？
未知命令怎么办？
客户端异常断开怎么办？
广播是否发给自己？
慢客户端是否会拖慢所有人？
```

如果这些问题没有答案，后续写代码时就会变成各种临时 if。

---

## 补充项目边界：第一版不做什么

为了保证项目能落地，第一版先不做：

```text
账号登录。
消息持久化。
离线消息。
多房间。
私聊加密。
分布式部署。
```

第一版只证明：

```text
TCP 长连接可管理。
消息边界清楚。
广播可靠执行。
连接退出能清理。
```

边界越清楚，项目越容易完成。

---

## 补充验收命令：设计阶段也要能验证

虽然本节主要是设计，但也要写出后续验收命令：

```bash
go run ./cmd/server
nc 127.0.0.1 9000
```

手动输入：

```text
/name alice
/users
hello
/quit
```

预期：

```text
昵称设置成功。
在线列表包含 alice。
普通文本能广播。
quit 后连接关闭。
```

把这些验收写在设计阶段，后续实现时才不会偏离目标。

---

## 五、常见问题

### 1. 为什么不直接用 JSON 作为协议？

可以用 JSON，但仍然需要消息边界。文本行协议更适合第一版练习。

### 2. 昵称是否应该由客户端完全决定？

不能完全信任客户端。服务端需要做去重、长度限制和非法字符过滤。

### 3. 聊天室项目为什么适合学 TCP？

它天然需要长连接、广播、连接清理和消息边界，能覆盖 TCP 编程核心问题。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 说清聊天室功能范围。
- 创建项目目录。
- 解释为什么需要协议边界。
- 解释 Client 和 Server 结构设计。
