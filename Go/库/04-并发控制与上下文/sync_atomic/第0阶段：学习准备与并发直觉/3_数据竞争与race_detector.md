# 3. 数据竞争与 race detector

本节目标：理解什么是数据竞争，学会使用 `go test -race` 发现并发读写问题，并知道 race detector 的边界。

数据竞争是 Go 并发编程里最需要警惕的问题之一。它经常不会立刻 panic，而是悄悄制造错误结果。

---

## 一、什么是数据竞争

当满足下面三个条件时，就可能出现数据竞争：

```text
两个或多个 goroutine 同时访问同一个变量；
至少有一个访问是写操作；
这些访问之间没有同步保护。
```

例如：

```go
var count int

go func() {
    count++
}()

go func() {
    count++
}()
```

两个 goroutine 都写 `count`，并且没有锁、channel、atomic 等同步手段，这就是数据竞争。

---

## 二、race detector 的基本用法

对测试运行竞态检测：

```bash
go test -race ./...
```

也可以直接运行某个程序：

```bash
go run -race main.go
```

如果发现数据竞争，通常会看到类似信息：

```text
WARNING: DATA RACE
Read at ...
Previous write at ...
Goroutine ... created at ...
```

重点看三类信息：

- 哪个变量被并发访问。
- 哪些代码位置发生读写。
- 相关 goroutine 是在哪里创建的。

---

## 三、一个典型 race 测试

```go
package counter

import "testing"

func TestRaceCounter(t *testing.T) {
    count := 0
    done := make(chan struct{})

    for i := 0; i < 1000; i++ {
        go func() {
            count++
            done <- struct{}{}
        }()
    }

    for i := 0; i < 1000; i++ {
        <-done
    }

    t.Logf("count=%d", count)
}
```

运行：

```bash
go test -race
```

你大概率会看到 race 报告。

注意：`done` channel 只保证主 goroutine 等待所有任务结束，并没有保护 `count++` 这段临界区。

---

## 四、修复思路一：Mutex

```go
var mu sync.Mutex
count := 0

go func() {
    mu.Lock()
    count++
    mu.Unlock()
    done <- struct{}{}
}()
```

锁让同一时刻只有一个 goroutine 能执行 `count++`。

---

## 五、修复思路二：atomic

```go
var count atomic.Int64

go func() {
    count.Add(1)
    done <- struct{}{}
}()
```

如果只是简单计数，atomic 通常更直接。

但是如果要同时保护多个字段之间的不变量，`Mutex` 往往更清晰。

---

## 六、race detector 不能证明什么

`go test -race` 很有用，但它不是数学证明。

它只能检测“当前执行路径中实际发生的数据竞争”。如果测试没有覆盖某段并发路径，就可能发现不了。

例如：

- 某个分支测试没有跑到。
- 并发压力太小，没有触发某种交错。
- 生产环境才有的请求模式，本地测试没有模拟。

所以正确姿势是：

```text
先设计清楚同步规则
再写并发测试覆盖关键路径
然后用 -race 辅助验证
```

不要反过来认为：

```text
跑了 -race 没报错，所以代码一定没问题
```

---

## 七、普通 map 的并发风险

Go 普通 map 不支持并发读写。

下面代码可能 panic，也可能被 race detector 报告：

```go
m := map[string]int{}

go func() {
    for {
        m["a"]++
    }
}()

go func() {
    for {
        _ = m["a"]
    }
}()
```

真实后端中，本地缓存如果直接用普通 map，就必须配合锁。

---

## 八、本节练习

练习一：运行不安全计数器，观察 `go test -race` 输出。

练习二：用 `sync.Mutex` 修复它。

练习三：用 `atomic.Int64` 再修复一版。

练习四：写一个普通 map 并发读写例子，观察 race 或 panic。

---

## 九、本节达标标准

学完本节后，你应该能做到：

- 准确定义数据竞争。
- 使用 `go test -race` 检测测试中的数据竞争。
- 能看懂 race 报告中的读写位置和 goroutine 创建位置。
- 知道 race detector 只能辅助验证，不能代替并发设计。
- 能说清普通 map 并发读写为什么危险。
