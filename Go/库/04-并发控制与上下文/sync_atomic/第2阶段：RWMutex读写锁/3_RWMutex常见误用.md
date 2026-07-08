# 3. RWMutex 常见误用

本节目标：识别 `RWMutex` 的常见错误，避免为了“读写锁看起来高级”而引入更复杂的并发 bug。

---

## 一、读锁期间写数据

错误示例：

```go
c.mu.RLock()
defer c.mu.RUnlock()

c.items[key] = value
```

读锁只允许读共享状态。写 map 必须使用写锁。

---

## 二、以为 RLock 不需要 Unlock

错误示例：

```go
c.mu.RLock()
value := c.items[key]
return value
```

忘记 `RUnlock` 会导致写锁一直等待读者释放。

推荐：

```go
c.mu.RLock()
defer c.mu.RUnlock()
```

---

## 三、读锁升级写锁

错误示例：

```go
c.mu.RLock()
if _, ok := c.items[key]; !ok {
    c.mu.Lock()
    c.items[key] = value
    c.mu.Unlock()
}
c.mu.RUnlock()
```

不要在持有读锁时申请写锁。

正确做法是释放读锁后申请写锁，并在写锁内重新检查条件。

---

## 四、写锁降级读锁

有些语言或库支持锁降级，但 Go 的 `RWMutex` 不提供直接锁降级语义。

不要写复杂的锁转换逻辑。通常更简单的做法是：

```text
明确划分读方法和写方法；
需要复杂状态迁移时，直接使用写锁完成整个关键操作。
```

---

## 五、盲目替换 Mutex

不是所有 `Mutex` 都应该换成 `RWMutex`。

`RWMutex` 有额外管理成本。如果临界区非常短，读写比例不极端，普通 `Mutex` 可能更快、更简单。

判断依据应该来自：

- benchmark。
- pprof。
- 实际锁竞争指标。
- 代码复杂度评估。

---

## 六、读操作中调用外部函数

即使是读锁，也不建议在锁内调用外部未知函数。

例如：

```go
c.mu.RLock()
defer c.mu.RUnlock()

return formatter(c.items)
```

如果 `formatter` 很慢，写者会长期等待。如果 `formatter` 反过来调用缓存写方法，可能引发死锁。

推荐：

```text
锁内复制必要数据；
解锁；
锁外执行格式化或慢操作。
```

---

## 七、本节练习

1. 写一个读锁期间写 map 的错误例子。
2. 写一个读锁升级写锁的错误例子。
3. 把一个 `Mutex` 缓存盲目改成 `RWMutex`，然后分析复杂度是否增加。
4. 给 `Snapshot` 增加锁外格式化逻辑。

---

## 八、本节达标标准

学完本节后，你应该能做到：

- 识别读锁写数据、忘记 `RUnlock`、锁升级等错误。
- 不盲目把 `Mutex` 换成 `RWMutex`。
- 能把慢操作移出读锁范围。
