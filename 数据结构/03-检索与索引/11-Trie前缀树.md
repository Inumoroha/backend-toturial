# 11. Trie 前缀树

本节目标：你能用 Trie 处理前缀匹配场景，比如路由、关键字和短链接前缀。

简短引入：Trie 可以理解为按字符一层层走的树。它适合处理“有没有某个前缀”“哪些词以这个前缀开头”这类问题。

## 一、为什么需要它

后端里常见前缀场景包括接口路由匹配、敏感词前缀、搜索建议、短链接 code 检查。用切片逐个扫描也能做，但数据量变大时会越来越慢。

```text
Trie 适合内存检索，不适合直接替代数据库全文搜索或专业搜索引擎。
```

## 二、基本用法

```powershell
mkdir ds-trie-demo; cd ds-trie-demo
go mod init ds-trie-demo
notepad main.go
go run .
```

```bash
mkdir ds-trie-demo && cd ds-trie-demo
go mod init ds-trie-demo
cat > main.go
go run .
```

`main.go`：

```go
package main

import "fmt"

type Node struct {
	children map[rune]*Node
	end      bool
}

type Trie struct {
	root *Node
}

func NewTrie() *Trie {
	return &Trie{root: &Node{children: map[rune]*Node{}}}
}

func (t *Trie) Insert(word string) {
	cur := t.root
	for _, ch := range word {
		if cur.children[ch] == nil {
			cur.children[ch] = &Node{children: map[rune]*Node{}}
		}
		cur = cur.children[ch]
	}
	cur.end = true
}

func (t *Trie) HasPrefix(prefix string) bool {
	cur := t.root
	for _, ch := range prefix {
		if cur.children[ch] == nil {
			return false
		}
		cur = cur.children[ch]
	}
	return true
}

func main() {
	trie := NewTrie()
	trie.Insert("/api/users")
	trie.Insert("/api/orders")
	fmt.Println(trie.HasPrefix("/api/u"))
}
```

## 三、关键参数/语法/代码结构

- `children` 保存下一层字符到节点的映射。
- `end` 表示一个完整词在这里结束。
- 使用 `rune` 能支持中文等 Unicode 字符。
- Trie 会占用较多内存，尤其是 key 很多且前缀不共享时。

## 四、真实后端场景示例

短链接服务可以用 Trie 快速判断某个 code 前缀是否已经被保留，比如 `admin`、`api`、`static`。生成短链接前先检查前缀，避免和系统路径冲突。

生产中仍要用数据库唯一索引保证 code 不重复，Trie 只能做提前判断和加速。

## 五、注意点

Trie 数据更新要考虑并发。配置热更新时，可以构建一棵新 Trie，验证通过后整体替换引用，避免边更新边读取造成不一致。

## 六、常见误区

- 误区一：把 Trie 当模糊搜索。Trie 擅长前缀，不擅长任意包含。
- 误区二：忽略内存成本。大量随机字符串共享前缀少，收益会下降。
- 误区三：不区分完整匹配和前缀匹配。`end` 字段就是为了解决这个问题。
- 误区四：只靠 Trie 保证唯一。服务重启和多实例下会失效。

## 七、本节达标标准

- 能实现 Trie 的插入和前缀判断。
- 能说明 Trie 适合路由、关键字、短链接前缀。
- 能区分完整匹配和前缀匹配。
- 能理解 Trie 与数据库唯一索引、搜索引擎的边界。
