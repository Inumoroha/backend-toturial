# 05. Split 与 Fields 处理标签和搜索词

本节目标：你能把标签、搜索词、权限列表这类字符串拆成可处理的切片，并过滤空值。

简短引入：后端接口经常收到列表型字符串，例如文章标签 `"go,backend,api"`、搜索词 `"go 后端 教程"`、权限列表 `"user:read,order:write"`。`Split` 和 `Fields` 用来把它们拆开。

## 一、为什么需要它

可以理解为：列表型输入进入业务逻辑前，要先从“一整段文本”变成“多个明确元素”。

常见场景是：

- 文章标签拆分。
- 搜索关键词拆分。
- 批量导入 ID 列表拆分。
- 配置项中多个开关拆分。

```text
拆分后的每个元素都仍然是用户输入，都要单独清洗和校验。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	fmt.Printf("%q\n", strings.Split("go,backend,api", ","))
	fmt.Printf("%q\n", strings.Fields("go   后端\t教程"))
}
```

Windows PowerShell：

```powershell
go run main.go
```

Linux/macOS：

```bash
go run main.go
```

这个命令适合观察拆分规则。真实项目中，拆完以后通常会继续去空、去重、限制数量。

## 三、关键代码结构

`strings.Split(s, sep)`：按固定分隔符拆分。

`strings.SplitN(s, sep, n)`：最多拆成 `n` 段，适合只关心前几段的场景。

`strings.Fields(s)`：按连续空白拆分，适合搜索词。

一个重要差异：

```go
strings.Split("go,,api", ",") // ["go", "", "api"]
strings.Fields("go   api")    // ["go", "api"]
```

`Split` 会保留中间空字段，`Fields` 会忽略连续空白。

## 四、真实后端场景示例

下面处理文章标签：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

func NormalizeTags(raw string) ([]string, error) {
	parts := strings.Split(raw, ",")
	seen := make(map[string]bool)
	tags := make([]string, 0, len(parts))

	for _, part := range parts {
		tag := strings.ToLower(strings.TrimSpace(part))
		if tag == "" {
			continue
		}
		if len(tag) > 20 {
			return nil, errors.New("tag too long")
		}
		if !seen[tag] {
			seen[tag] = true
			tags = append(tags, tag)
		}
	}

	if len(tags) > 5 {
		return nil, errors.New("too many tags")
	}
	return tags, nil
}

func main() {
	tags, err := NormalizeTags(" Go, backend,go, API, ")
	if err != nil {
		fmt.Println("标签错误:", err)
		return
	}
	fmt.Printf("%q\n", tags)
}
```

真实项目中，标签入库前还要考虑唯一索引和事务边界。比如创建文章和创建标签关联表，通常应该放在同一个事务里，避免文章创建成功但标签关联失败。

## 五、注意点

拆分列表时要考虑四个边界：

- 空字符串。
- 连续分隔符。
- 数量上限。
- 单个元素长度上限。

对于搜索词，还要考虑索引成本。搜索词越多，拼出的查询条件越复杂，可能影响数据库性能。

```text
搜索接口要限制关键词长度和数量，不能让用户输入无限扩张成慢查询。
```

## 六、常见误区

误区一：`Split` 后直接使用每个元素。

元素两边可能有空格，也可能为空，必须逐个清洗。

误区二：不限制标签数量。

标签无限制会影响数据库写入、索引维护和页面展示。

误区三：把搜索词直接拼成 SQL。

应使用参数化查询。动态字段也要走白名单映射。

## 七、本节达标标准

- 能用 `Split` 拆分逗号列表。
- 能用 `Fields` 拆分搜索词。
- 能处理空元素、重复元素、数量限制。
- 能说明列表拆分后仍需逐项校验。

