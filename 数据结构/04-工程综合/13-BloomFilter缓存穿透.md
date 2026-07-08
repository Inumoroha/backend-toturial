# 13. Bloom Filter 缓存穿透

本节目标：你能用 Bloom Filter 做缓存穿透的第一层防护，并理解它的误判边界。

简短引入：缓存穿透指请求一直查询不存在的数据，缓存没有命中，数据库又不断被打。Bloom Filter 可以先判断“这个 key 是否可能存在”，明显不存在的请求直接拦截。

## 一、为什么需要它

短链接服务中，攻击者可能不断请求随机 code。缓存查不到，数据库也查不到，但每次都打到数据库。Bloom Filter 可以把已存在 code 加进去，请求进来先判断，不可能存在就直接返回 404。

```text
Bloom Filter 可能误判存在，但不能误判不存在。
```

## 二、基本用法

```powershell
mkdir ds-bloom-demo; cd ds-bloom-demo
go mod init ds-bloom-demo
notepad main.go
go run .
```

```bash
mkdir ds-bloom-demo && cd ds-bloom-demo
go mod init ds-bloom-demo
cat > main.go
go run .
```

`main.go`：

```go
package main

import (
	"fmt"
	"hash/fnv"
)

type Bloom struct {
	bits []bool
	size uint64
}

func NewBloom(size uint64) *Bloom {
	return &Bloom{bits: make([]bool, size), size: size}
}

func hash(s string, seed byte) uint64 {
	h := fnv.New64a()
	h.Write([]byte{seed})
	h.Write([]byte(s))
	return h.Sum64()
}

func (b *Bloom) Add(s string) {
	b.bits[hash(s, 1)%b.size] = true
	b.bits[hash(s, 2)%b.size] = true
}

func (b *Bloom) MayExist(s string) bool {
	return b.bits[hash(s, 1)%b.size] && b.bits[hash(s, 2)%b.size]
}

func main() {
	b := NewBloom(1024)
	b.Add("abc123")
	fmt.Println(b.MayExist("abc123"))
	fmt.Println(b.MayExist("not-exist"))
}
```

## 三、关键参数/语法/代码结构

- `bits` 是位数组，这里用 `[]bool` 简化。
- 多个 hash 位置都为 true，才认为“可能存在”。
- 返回 false 时可以确定不存在。
- 返回 true 时只是可能存在，还要查缓存或数据库。

## 四、真实后端场景示例

短链接读取流程可以是：

1. 请求 `/s/{code}`。
2. Bloom Filter 判断不存在，直接返回 404。
3. 可能存在，查 Redis。
4. Redis 未命中，再查数据库。
5. 数据库命中后回填 Redis。

数据库仍然需要唯一索引保证 code 唯一。Bloom Filter 只是挡掉大量明显无效请求。

## 五、注意点

删除是 Bloom Filter 的难点。普通 Bloom Filter 不适合频繁删除 key。真实项目中可以定期重建，或使用 Counting Bloom Filter，但实现复杂度会提高。

## 六、常见误区

- 误区一：以为 Bloom Filter 说存在就一定存在。它只能说可能存在。
- 误区二：把 Bloom Filter 当权限判断。误判会带来安全风险。
- 误区三：容量太小，误判率升高，数据库压力仍然大。
- 误区四：新增数据后忘记同步加入过滤器。

## 七、本节达标标准

- 能说清 Bloom Filter 的“不存在一定不存在，存在只是可能存在”。
- 能写一个简化版 Bloom Filter。
- 能把它放到短链接缓存穿透场景里。
- 能理解容量、误判、删除和同步更新的风险。
