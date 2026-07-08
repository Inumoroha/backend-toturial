# 9. Fuzz 测试：发现解析器的边界输入

本节目标：学完后你能用 fuzz 测试发现解析函数在异常输入下的崩溃和边界问题。

简短引入：后端服务会处理大量用户输入，例如 URL、JSON 字段、订单号、邀请码、权限表达式。普通测试只能覆盖你想到的输入，fuzz 会自动生成大量输入，帮你发现没想到的边界。

## 一、为什么需要它

假设你写了一个短链接 slug 校验函数。你可能会测：

- `abc123` 合法。
- 空字符串非法。
- 太长非法。
- 有空格非法。

但真实用户输入可能包含控制字符、中文、超长字符串、奇怪符号。fuzz 的价值是让工具帮你不断尝试这些输入。

```text
Fuzz 适合测试“输入空间很大”的解析、校验、编解码逻辑。
```

它不适合直接拿来测依赖数据库、网络或外部状态的代码。

## 二、基本用法

创建项目：

Windows PowerShell：

```powershell
mkdir fuzzdemo
cd fuzzdemo
go mod init example.com/fuzzdemo
```

Linux/macOS：

```bash
mkdir fuzzdemo
cd fuzzdemo
go mod init example.com/fuzzdemo
```

新建 `slug.go`：

```go
package fuzzdemo

func IsValidSlug(s string) bool {
	if len(s) < 3 || len(s) > 32 {
		return false
	}
	for i := 0; i < len(s); i++ {
		c := s[i]
		if !((c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '-') {
			return false
		}
	}
	return true
}
```

新建 `slug_test.go`：

```go
package fuzzdemo

import "testing"

func TestIsValidSlug(t *testing.T) {
	tests := []struct {
		s    string
		want bool
	}{
		{s: "abc", want: true},
		{s: "a", want: false},
		{s: "abc-123", want: true},
		{s: "ABC", want: false},
		{s: "abc def", want: false},
	}

	for _, tt := range tests {
		got := IsValidSlug(tt.s)
		if got != tt.want {
			t.Fatalf("IsValidSlug(%q) = %v, want %v", tt.s, got, tt.want)
		}
	}
}

func FuzzIsValidSlug(f *testing.F) {
	f.Add("abc")
	f.Add("abc-123")
	f.Add("ABC")
	f.Add("")

	f.Fuzz(func(t *testing.T, s string) {
		got := IsValidSlug(s)
		if got {
			if len(s) < 3 || len(s) > 32 {
				t.Fatalf("valid slug has invalid length: %q", s)
			}
			for i := 0; i < len(s); i++ {
				c := s[i]
				ok := (c >= 'a' && c <= 'z') || (c >= '0' && c <= '9') || c == '-'
				if !ok {
					t.Fatalf("valid slug contains invalid char: %q", s)
				}
			}
		}
	})
}
```

运行普通测试：

```powershell
go test ./...
```

Linux/macOS：

```bash
go test ./...
```

运行 fuzz：

```powershell
go test -fuzz=FuzzIsValidSlug -fuzztime=10s
```

Linux/macOS：

```bash
go test -fuzz=FuzzIsValidSlug -fuzztime=10s
```

## 三、关键参数/语法/代码结构

fuzz 函数以 `Fuzz` 开头，参数是 `*testing.F`。

`f.Add` 添加种子输入。种子可以理解为给 fuzz 的起点，应该包含典型合法和非法值。

`f.Fuzz(func(t *testing.T, s string) { ... })` 是 fuzz 要反复执行的函数。

`-fuzz=FuzzIsValidSlug` 指定运行哪个 fuzz 测试。

`-fuzztime=10s` 控制运行时间。真实项目中，CI 可以短一点，本地排查可以长一点。

## 四、真实后端场景示例

短链接系统中，slug 最终可能出现在 URL 路径里。我们希望只允许小写字母、数字和短横线，不允许 `/`、空格、问号等字符。

普通单元测试负责写清楚你知道的规则。fuzz 负责帮你找奇怪输入。

可以再加一个性质：如果 `IsValidSlug(s)` 返回 true，那么它应该可以安全拼到路径后面：

```go
func FuzzSlugPathSafe(f *testing.F) {
	f.Add("abc-123")

	f.Fuzz(func(t *testing.T, s string) {
		if !IsValidSlug(s) {
			return
		}
		for i := 0; i < len(s); i++ {
			if s[i] == '/' || s[i] == '?' || s[i] == '#' {
				t.Fatalf("slug is not path safe: %q", s)
			}
		}
	})
}
```

这类测试很适合解析器、校验器、编码器。比如：

- 解析订单号。
- 校验邀请码。
- 解析分页参数。
- 解析权限表达式。
- URL 白名单校验。

## 五、注意点

fuzz 测试必须有明确性质。不要只写“函数跑一下不 panic”，除非这个函数本身就是解析不可信输入的底线防护。

不要在 fuzz 里访问数据库、Redis、HTTP 接口。fuzz 会大量调用目标函数，这些外部依赖会让测试非常慢，也可能产生大量脏数据。

fuzz 找到失败输入后，Go 会保存触发失败的样本。修复 bug 后，这个样本会变成回归测试资产。

如果 fuzz 很慢，先缩小目标函数。比如只测 slug 校验，不测完整短链接创建流程。

## 六、常见误区

- 没有写 `f.Add` 种子：fuzz 仍能跑，但起点不贴近业务，效率会差。
- 在 fuzz 中依赖当前时间或随机数：结果不可重复，失败难排查。
- 让 fuzz 写数据库：容易制造大量脏数据。
- 把 fuzz 当成替代单元测试：fuzz 找边界，普通测试表达业务规则，两者都需要。
- 性质写得太宽泛：例如只检查“不崩溃”，可能发现不了业务错误。

## 七、本节达标标准

- 能写 `FuzzXxx(f *testing.F)`。
- 能添加合法和非法种子输入。
- 能运行 `go test -fuzz=... -fuzztime=...`。
- 能为解析或校验函数设计基本性质。
- 能说明为什么 fuzz 不适合直接访问外部资源。

