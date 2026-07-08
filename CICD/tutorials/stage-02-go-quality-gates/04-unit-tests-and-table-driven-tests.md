# 04：单元测试与表格驱动测试

## 1. 本节目标

这一节学习 Go 测试基础。

你要掌握：

- 测试文件命名。
- 测试函数命名。
- `go test ./...`。
- 表格驱动测试。
- 测试错误信息怎么写。
- 如何组织 service 层测试。

## 2. Go 测试文件规则

测试文件必须以 `_test.go` 结尾。

例如：

```text
internal/service/todo.go
internal/service/todo_test.go
```

测试函数格式：

```go
func TestXxx(t *testing.T) {
}
```

示例：

```go
package service

import "testing"

func TestAdd(t *testing.T) {
	got := 1 + 1
	want := 2

	if got != want {
		t.Fatalf("got %d, want %d", got, want)
	}
}
```

运行当前包测试：

```bash
go test
```

运行整个项目：

```bash
go test ./...
```

## 3. 测试命名

推荐命名表达行为：

```go
func TestCreateTodoRejectsEmptyTitle(t *testing.T) {}
func TestCreateTodoUsesDefaultPriority(t *testing.T) {}
func TestCompleteTodoReturnsNotFound(t *testing.T) {}
```

好的测试名应该让人不看实现也知道在验证什么。

## 4. 表格驱动测试

表格驱动测试是 Go 项目非常常见的写法。

假设有一个函数：

```go
package service

import "strings"

func NormalizeTitle(title string) string {
	return strings.TrimSpace(title)
}
```

测试：

```go
package service

import "testing"

func TestNormalizeTitle(t *testing.T) {
	tests := []struct {
		name  string
		input string
		want  string
	}{
		{
			name:  "keeps normal title",
			input: "buy milk",
			want:  "buy milk",
		},
		{
			name:  "trims spaces",
			input: "  buy milk  ",
			want:  "buy milk",
		},
		{
			name:  "handles empty string",
			input: "",
			want:  "",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := NormalizeTitle(tt.input)
			if got != tt.want {
				t.Fatalf("NormalizeTitle(%q) = %q, want %q", tt.input, got, tt.want)
			}
		})
	}
}
```

优点：

- 多个场景结构统一。
- 新增 case 很方便。
- 失败时能看到具体 case 名字。

## 5. t.Helper

如果你写测试辅助函数，建议使用 `t.Helper()`。

```go
func requireEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()

	if got != want {
		t.Fatalf("got %v, want %v", got, want)
	}
}
```

`t.Helper()` 可以让失败行号指向调用辅助函数的位置，而不是辅助函数内部。

## 6. service 层测试示例

假设业务规则：

```text
创建 todo 时 title 不能为空。
priority 为空时默认 normal。
```

代码：

```go
package service

import (
	"errors"
	"strings"
)

var ErrEmptyTitle = errors.New("empty title")

type Todo struct {
	Title    string
	Priority string
}

func NewTodo(title string, priority string) (Todo, error) {
	title = strings.TrimSpace(title)
	if title == "" {
		return Todo{}, ErrEmptyTitle
	}
	if priority == "" {
		priority = "normal"
	}
	return Todo{Title: title, Priority: priority}, nil
}
```

测试：

```go
package service

import (
	"errors"
	"testing"
)

func TestNewTodo(t *testing.T) {
	tests := []struct {
		name         string
		title        string
		priority     string
		wantTitle    string
		wantPriority string
		wantErr      error
	}{
		{
			name:         "uses explicit priority",
			title:        "buy milk",
			priority:     "high",
			wantTitle:    "buy milk",
			wantPriority: "high",
		},
		{
			name:         "uses default priority",
			title:        "buy milk",
			priority:     "",
			wantTitle:    "buy milk",
			wantPriority: "normal",
		},
		{
			name:    "rejects empty title",
			title:   "   ",
			wantErr: ErrEmptyTitle,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := NewTodo(tt.title, tt.priority)
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("error = %v, want %v", err, tt.wantErr)
			}
			if tt.wantErr != nil {
				return
			}
			if got.Title != tt.wantTitle {
				t.Fatalf("title = %q, want %q", got.Title, tt.wantTitle)
			}
			if got.Priority != tt.wantPriority {
				t.Fatalf("priority = %q, want %q", got.Priority, tt.wantPriority)
			}
		})
	}
}
```

## 7. 常用 go test 参数

运行所有测试：

```bash
go test ./...
```

显示详细输出：

```bash
go test -v ./...
```

只运行某个测试：

```bash
go test -run TestNewTodo ./internal/service
```

禁用测试缓存：

```bash
go test -count=1 ./...
```

运行某个子测试：

```bash
go test -run 'TestNewTodo/rejects_empty_title' ./internal/service
```

## 8. 测试错误信息怎么写

推荐：

```go
t.Fatalf("NewTodo(%q, %q) error = %v, want %v", title, priority, err, wantErr)
```

不推荐：

```go
t.Fatal("failed")
```

好的错误信息应该说明：

- 调用了什么。
- 得到了什么。
- 期望什么。

## 9. 小练习

在你的项目中创建：

```text
internal/service/todo.go
internal/service/todo_test.go
```

实现并测试：

```go
func NormalizeTitle(title string) string
func ValidateTitle(title string) error
func NormalizePriority(priority string) string
```

要求：

- 使用表格驱动测试。
- 每个函数至少 3 个 case。
- 执行 `go test ./...` 通过。

## 10. 本节小结

你现在应该理解：

- Go 测试文件以 `_test.go` 结尾。
- 测试函数以 `TestXxx` 命名。
- `go test ./...` 是最重要的质量门禁之一。
- 表格驱动测试适合覆盖多种输入场景。
- service 层是初学者最适合开始补测试的地方。

