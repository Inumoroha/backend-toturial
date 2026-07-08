# 6. 链表与 LRU

本节目标：你能理解链表在缓存淘汰中的作用，并用 Go 标准库写一个小型 LRU 缓存。

简短引入：链表单独在业务代码里不常见，但它和 map 组合后非常实用。LRU 缓存就是典型例子：最近使用的数据放前面，最久没用的数据淘汰。

## 一、为什么需要它

缓存容量有限，不能无限放。真实项目中常见场景是缓存用户资料、权限配置、热点短链接。如果缓存满了，LRU 会优先淘汰最久没访问的数据。

```text
缓存只能提升读取速度，不能成为唯一数据源。
```

## 二、基本用法

```powershell
mkdir ds-lru-demo; cd ds-lru-demo
go mod init ds-lru-demo
notepad main.go
go run .
```

```bash
mkdir ds-lru-demo && cd ds-lru-demo
go mod init ds-lru-demo
cat > main.go
go run .
```

`main.go`：

```go
package main

import (
	"container/list"
	"fmt"
)

type entry struct {
	key   string
	value string
}

type LRU struct {
	capacity int
	ll       *list.List
	cache    map[string]*list.Element
}

func NewLRU(capacity int) *LRU {
	return &LRU{capacity: capacity, ll: list.New(), cache: make(map[string]*list.Element)}
}

func (l *LRU) Get(key string) (string, bool) {
	if ele, ok := l.cache[key]; ok {
		l.ll.MoveToFront(ele)
		return ele.Value.(entry).value, true
	}
	return "", false
}

func (l *LRU) Put(key, value string) {
	if ele, ok := l.cache[key]; ok {
		ele.Value = entry{key: key, value: value}
		l.ll.MoveToFront(ele)
		return
	}
	ele := l.ll.PushFront(entry{key: key, value: value})
	l.cache[key] = ele
	if l.ll.Len() > l.capacity {
		back := l.ll.Back()
		l.ll.Remove(back)
		delete(l.cache, back.Value.(entry).key)
	}
}

func main() {
	lru := NewLRU(2)
	lru.Put("u:1", "Alice")
	lru.Put("u:2", "Bob")
	lru.Get("u:1")
	lru.Put("u:3", "Chen")
	fmt.Println(lru.Get("u:2"))
	fmt.Println(lru.Get("u:1"))
}
```

## 三、关键参数/语法/代码结构

- map 负责 O(1) 找到节点。
- 双向链表负责快速移动节点和删除尾部。
- `MoveToFront` 表示刚访问过，放到最前。
- `Back` 是最久未使用的节点，容量超限时删除它。

## 四、真实后端场景示例

短链接服务读取 `code -> longURL` 很频繁，可以在进程内放一个小 LRU。缓存命中时直接返回，未命中时查数据库，再写入 LRU。

注意，生产环境要考虑多实例。每个实例的内存缓存不同步，所以最终一致性要靠数据库或 Redis 这类集中式缓存兜底。

## 五、注意点

这个 LRU 不是并发安全的。真实 HTTP 服务里多个请求会同时读写缓存，需要加 `sync.Mutex`，或者使用成熟缓存库。

```text
任何会被多个 goroutine 同时读写的内存结构，都要明确并发保护策略。
```

## 六、常见误区

- 误区一：只用 map 做缓存，没有淘汰策略，内存持续增长。
- 误区二：LRU 不加锁就放进 HTTP 服务。
- 误区三：缓存删除不考虑数据库事务。事务回滚但缓存已更新，会造成脏读。
- 误区四：缓存 key 设计混乱。不同业务共用 key 前缀容易互相覆盖。

## 七、本节达标标准

- 能解释 map 加双向链表为什么能实现 LRU。
- 能用 `container/list` 写一个小型 LRU。
- 能说明 LRU 在用户资料、权限、短链接缓存中的作用。
- 能识别并发安全、缓存一致性和容量控制风险。
