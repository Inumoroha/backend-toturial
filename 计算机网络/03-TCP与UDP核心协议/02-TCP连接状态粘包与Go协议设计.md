# 02-TCP 连接状态、粘包与 Go 协议设计

## 学习目标

理解 TCP 关闭连接、TIME_WAIT、CLOSE_WAIT、粘包拆包，并学会设计一个简单的应用层协议。

## 一、四次挥手

TCP 连接是双向的，关闭时每个方向都要单独关闭。

典型流程：

```text
主动关闭方 -> 被动关闭方：FIN
被动关闭方 -> 主动关闭方：ACK
被动关闭方 -> 主动关闭方：FIN
主动关闭方 -> 被动关闭方：ACK
```

所以通常是四次挥手。

## 二、TIME_WAIT

主动关闭连接的一方通常进入 TIME_WAIT。

TIME_WAIT 的作用：

- 确保最后一个 ACK 能被对方收到。
- 避免旧连接的延迟报文影响新连接。

大量 TIME_WAIT 常见于：

- 短连接频繁创建和关闭。
- HTTP Client 没有复用连接。
- 代理或网关层连接策略不合理。

查看：

```bash
ss -ant state time-wait
```

## 三、CLOSE_WAIT

CLOSE_WAIT 通常出现在被动关闭方。

如果大量 CLOSE_WAIT，常见原因是程序收到对方关闭后，没有关闭本地 socket。

Go 中常见原因：

- HTTP Client 没有调用 `resp.Body.Close()`。
- 自己写 TCP 程序时没有在读到 EOF 或错误后关闭连接。
- goroutine 卡住，连接生命周期没有结束。

查看：

```bash
ss -ant state close-wait
```

## 四、TCP 是字节流

TCP 不保留应用层消息边界。

你写入两次：

```text
hello
world
```

接收方可能读到：

```text
helloworld
```

也可能读到：

```text
hel
lowor
ld
```

这不是 TCP 错误，而是 TCP 的字节流特性。

## 五、粘包与拆包

所谓粘包：

- 发送方多次写入的数据，被接收方一次读到。

所谓拆包：

- 发送方一次写入的数据，被接收方多次读到。

解决方法不是“禁止粘包”，而是在应用层定义消息边界。

## 六、常见协议边界设计

### 固定长度

每条消息固定 128 字节。简单但浪费空间，不灵活。

### 分隔符

每条消息以 `\n` 结尾。适合文本协议，例如 Redis 协议部分设计。

### 长度前缀

消息前 4 字节表示 body 长度。适合二进制协议和 JSON body。

## 七、Go 示例：长度前缀协议

写入消息：

```go
func writeFrame(w io.Writer, payload []byte) error {
    var header [4]byte
    binary.BigEndian.PutUint32(header[:], uint32(len(payload)))

    if _, err := w.Write(header[:]); err != nil {
        return err
    }
    _, err := w.Write(payload)
    return err
}
```

读取消息：

```go
func readFrame(r io.Reader) ([]byte, error) {
    var header [4]byte
    if _, err := io.ReadFull(r, header[:]); err != nil {
        return nil, err
    }

    n := binary.BigEndian.Uint32(header[:])
    if n > 1024*1024 {
        return nil, fmt.Errorf("frame too large: %d", n)
    }

    payload := make([]byte, n)
    if _, err := io.ReadFull(r, payload); err != nil {
        return nil, err
    }
    return payload, nil
}
```

关键点：

- `io.ReadFull` 确保读满指定长度。
- 必须限制最大消息长度，避免内存被打爆。
- 协议边界由应用层自己定义。

## 八、实验：观察连接状态

启动 HTTP 服务后，用循环请求制造连接：

```bash
for i in {1..100}; do curl -s http://127.0.0.1:8080 > /dev/null; done
ss -ant | grep 8080
```

观察是否出现 TIME_WAIT。

## 九、练习题

1. TIME_WAIT 为什么通常在主动关闭方？
2. 大量 CLOSE_WAIT 说明什么？
3. TCP 为什么会有粘包和拆包？
4. 长度前缀协议为什么要限制最大 body 大小？

## 十、验收标准

你能解释 TCP 字节流特性，并能写出基于长度前缀的消息读取逻辑。

