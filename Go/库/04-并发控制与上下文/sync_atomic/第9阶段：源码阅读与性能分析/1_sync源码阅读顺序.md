# 1. sync 源码阅读顺序

本节目标：建立一条适合后端工程师的 `sync` 源码阅读路线，避免一上来陷入 runtime 细节。

---

## 一、先读文档注释

源码文件通常在 Go 安装目录：

```text
$GOROOT/src/sync/
```

优先打开：

```text
mutex.go
rwmutex.go
waitgroup.go
once.go
map.go
pool.go
cond.go
```

第一遍只读每个类型的注释，重点关注：

- 这个类型解决什么问题。
- 是否允许复制。
- 方法的并发语义。
- panic 条件。
- 使用限制。

---

## 二、再读测试

标准库测试通常比源码更容易理解行为边界。

关注：

- 正常使用示例。
- 并发压力测试。
- 错误用法是否 panic。
- 边界场景。

如果能读懂测试，你对 API 行为的理解会稳很多。

---

## 三、第三步看核心字段

例如 `Mutex` 中会有表示状态的字段。

你不需要第一天就记住每一位含义，但要知道：

```text
同步原语内部通常会维护状态；
阻塞和唤醒可能依赖 runtime 信号量；
性能优化往往来自快路径和慢路径分离。
```

---

## 四、最后看 runtime 关联

进阶时再看：

- semaphore。
- goroutine park/unpark。
- scheduler。
- race detector hooks。

这一层更底层，不适合作为第一遍学习重点。

---

## 五、推荐阅读顺序

```text
WaitGroup
-> Once
-> Mutex
-> RWMutex
-> Cond
-> Map
-> Pool
```

原因：

- `WaitGroup`、`Once` 行为相对直观。
- `Mutex`、`RWMutex` 涉及锁状态和调度。
- `Map`、`Pool` 内部优化更多，适合后面看。

---

## 六、本节练习

1. 找到本机 `$GOROOT/src/sync`。
2. 阅读 `waitgroup.go` 的注释和 panic 条件。
3. 阅读 `once.go`，记录 panic 后行为。
4. 阅读 `map.go` 注释，记录适用场景。
5. 写一份源码阅读笔记。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 知道源码阅读先看注释和测试。
- 能找到核心文件。
- 能从源码注释中提炼 API 限制。
