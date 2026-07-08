# 03. Contains 与前后缀判断

本节目标：你能判断字符串是否包含关键词、是否有指定前缀或后缀，并把这些判断用于请求头、文件名和权限码处理。

简短引入：后端经常要先判断一个字符串是不是符合约定，再决定是否继续处理。`Contains`、`HasPrefix`、`HasSuffix` 是最常用的三个判断函数。

## 一、为什么需要它

可以理解为：这些函数负责回答“这个字符串像不像我要处理的东西”。

常见场景是：

- 请求头是否以 `Bearer ` 开头。
- 上传文件名是否以 `.jpg` 或 `.png` 结尾。
- 权限码是否包含某个模块名。
- 日志行是否包含错误关键字。

```text
前后缀判断适合做格式分流，不适合替代完整的安全校验。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	auth := "Bearer abc.def.ghi"

	fmt.Println(strings.HasPrefix(auth, "Bearer "))
	fmt.Println(strings.Contains(auth, "."))
	fmt.Println(strings.HasSuffix("avatar.png", ".png"))
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

这个命令能快速验证判断条件。真实项目中，这些判断通常位于中间件、上传处理、权限校验的入口处。

## 三、关键代码结构

`strings.Contains(s, substr)`：判断 `s` 是否包含 `substr`。

`strings.HasPrefix(s, prefix)`：判断 `s` 是否以 `prefix` 开头。

`strings.HasSuffix(s, suffix)`：判断 `s` 是否以 `suffix` 结尾。

返回值都是 `bool`，适合放在 `if` 里。

真实项目中，建议把判断顺序写得像一条小流水线：

1. 先 `TrimSpace` 清理外层空白。
2. 再判断前缀、后缀或包含关系。
3. 如果通过，再进入更严格的解析或校验。
4. 如果失败，尽早返回清晰错误。

这样做的好处是错误位置明确。比如认证头缺少 `Bearer `，就不要继续解析 token；上传文件后缀不在允许列表，也不要继续写磁盘。

## 四、真实后端场景示例

下面模拟从请求头中判断是否是 Bearer token：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

func ExtractBearerToken(authHeader string) (string, error) {
	authHeader = strings.TrimSpace(authHeader)
	if !strings.HasPrefix(authHeader, "Bearer ") {
		return "", errors.New("missing bearer token")
	}

	token := strings.TrimSpace(strings.TrimPrefix(authHeader, "Bearer "))
	if token == "" {
		return "", errors.New("empty bearer token")
	}
	return token, nil
}

func main() {
	token, err := ExtractBearerToken(" Bearer abc.def.ghi ")
	if err != nil {
		fmt.Println("认证失败:", err)
		return
	}
	fmt.Println("token:", token)
}
```

真实项目中，提取 token 后还要做签名校验、过期时间校验、用户状态校验。字符串前缀判断只负责“取出看起来像 token 的部分”。

再看一个上传文件名初筛的例子：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

func CheckImageFilename(filename string) error {
	name := strings.ToLower(strings.TrimSpace(filename))
	if name == "" {
		return errors.New("filename is required")
	}
	if !(strings.HasSuffix(name, ".jpg") || strings.HasSuffix(name, ".jpeg") || strings.HasSuffix(name, ".png")) {
		return errors.New("only jpg and png are allowed")
	}
	return nil
}

func main() {
	if err := CheckImageFilename(" Avatar.PNG "); err != nil {
		fmt.Println("上传失败:", err)
		return
	}
	fmt.Println("文件名初筛通过")
}
```

注意这里说的是“初筛”。真实上传流程还要检查文件内容、大小、MIME 类型，并且服务端应生成新的存储文件名。

## 五、注意点

文件后缀判断要特别保守。`HasSuffix(filename, ".jpg")` 只能说明名字看起来像图片，不能证明内容真的是图片。

上传文件真实项目中通常还需要：

- 限制文件大小。
- 检查 MIME 类型。
- 随机生成存储文件名。
- 不把用户传入的文件名直接作为服务器路径。

还有一个判断标准：如果你需要确认“完整格式是否正确”，不要只靠 `Contains`。比如邮箱不能只判断是否包含 `@`，URL 不能只判断是否包含 `http`。这类场景应该使用更合适的解析器或校验规则。

## 六、常见误区

误区一：用 `Contains(token, "Bearer")` 判断认证头。

这会接受 `"xxx Bearer yyy"` 这种不符合约定的输入。请求头协议通常应该用 `HasPrefix`。

误区二：用文件后缀判断文件安全。

攻击者可以把脚本文件改名成 `.jpg`。后缀判断只是第一层筛选。

误区三：忘记清理空白。

请求头、配置项、导入文件都可能有额外空白。判断前通常先 `TrimSpace`。

## 七、本节达标标准

- 能用 `Contains` 判断关键字是否存在。
- 能用 `HasPrefix` 处理协议前缀。
- 能用 `HasSuffix` 做文件名初筛。
- 能说明前后缀判断不能替代安全校验。
