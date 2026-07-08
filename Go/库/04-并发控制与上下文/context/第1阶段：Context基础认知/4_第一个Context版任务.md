# 4. 第一个 Context 版任务

本节目标：写出第一个真正支持 `context` 的任务函数。

很多初学者会创建 context，但函数内部没有使用它。这样的 context 没有实际意义。

真正支持 context 的函数应该做到：

```text
接收 ctx。
在可能阻塞或耗时的地方监听 ctx.Done()。
取消时返回 ctx.Err()。
```

---

## 一、先写一个不支持取消的函数

```go
func slowWork() error {
	time.Sleep(2 * time.Second)
	fmt.Println("work done")
	return nil
}
```

这个函数的问题是：一旦开始执行，就必须等 2 秒。

调用方没有办法中途取消。

---

## 二、改造成支持 Context

```go
func slowWork(ctx context.Context) error {
	select {
	case <-time.After(2 * time.Second):
		fmt.Println("work done")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

现在调用方可以传入一个会取消或超时的 context。

---

## 三、完整代码

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func slowWork(ctx context.Context) error {
	select {
	case <-time.After(2 * time.Second):
		fmt.Println("work done")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	err := slowWork(ctx)
	if errors.Is(err, context.DeadlineExceeded) {
		fmt.Println("work timeout")
		return
	}
	if err != nil {
		fmt.Println("work error:", err)
		return
	}

	fmt.Println("success")
}
```

运行结果：

```text
work timeout
```

---

## 四、把超时改长

把超时时间改成 3 秒：

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
```

再次运行，会看到：

```text
work done
success
```

这说明任务在 context 超时前完成了。

---

## 五、不要吞掉 ctx.Err()

不推荐：

```go
case <-ctx.Done():
	return errors.New("failed")
```

这样调用方无法知道是取消还是超时。

推荐：

```go
case <-ctx.Done():
	return ctx.Err()
```

如果需要添加上下文信息，可以包装错误：

```go
return fmt.Errorf("slow work stopped: %w", ctx.Err())
```

调用方仍然可以：

```go
errors.Is(err, context.DeadlineExceeded)
```

---

## 六、本节练习

请完成：

1. 写一个 `download(ctx context.Context)` 函数，模拟下载耗时 3 秒。
2. 使用 1 秒 timeout 调用它。
3. 使用 5 秒 timeout 调用它。
4. 取消时返回 `ctx.Err()`。
5. 在 main 中用 `errors.Is` 区分是否超时。

---

## 七、本阶段总结

你已经完成了 `context` 基础认知：

- `Context` 是接口。
- `Background` 和 `TODO` 是根 context。
- `Done` 用来接收取消信号。
- `Err` 用来判断取消原因。
- `Deadline` 用来读取截止时间。
- `Value` 用来读取请求级元数据。

下一阶段开始学习主动取消：`context.WithCancel`。

---

## 八、完整运行步骤

创建：

```text
cmd/01-first-context-task/main.go
```

运行：

```powershell
go run ./cmd/01-first-context-task
```

先让任务超时，再把 timeout 改长观察成功。

这个练习很重要，因为它是所有后续 context 代码的最小模型：

```text
任务完成。
ctx 取消。
二者谁先发生。
```

---

## 九、把任务改成循环型

很多真实任务不是一个 `time.After`，而是循环处理。

```go
func loopWork(ctx context.Context) error {
	for i := 1; i <= 10; i++ {
		select {
		case <-ctx.Done():
			return fmt.Errorf("loop work stopped: %w", ctx.Err())
		case <-time.After(200 * time.Millisecond):
			fmt.Println("step", i)
		}
	}
	return nil
}
```

这种写法比在循环里简单 `time.Sleep` 更容易响应取消。

---

## 十、常见错误

### 1. 函数接收 ctx 但不用

```go
func slowWork(ctx context.Context) error {
	time.Sleep(2 * time.Second)
	return nil
}
```

这只是形式上支持 context。

### 2. 取消时返回 nil

```go
case <-ctx.Done():
	return nil
```

调用方会以为任务成功。

### 3. 不包装业务上下文

直接返回 `ctx.Err()` 可以，但复杂项目里建议包装操作名。

---

## 十一、本节练习加强

写一个 `processFiles(ctx, files []string)`：

1. 每个文件处理 300ms。
2. 处理前打印文件名。
3. ctx 取消后停止处理后续文件。
4. 返回包装后的 `ctx.Err()`。
5. main 中设置 1 秒 timeout，观察处理了几个文件。
