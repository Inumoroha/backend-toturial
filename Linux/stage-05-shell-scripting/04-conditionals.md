# 04：条件判断

本节目标：学会使用 `if` 判断字符串、数字、文件和目录。

本节语法/命令顺序：

```text
1. if ... then ... fi
2. [ condition ]
3. [ -z "$var" ]
4. [ -f "$file" ]
5. [ -d "$dir" ]
6. [ "$a" = "$b" ]
7. [ "$n" -gt 10 ]
8. [[ condition ]]
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-05
```

---

## 1. 最小 if 语句

创建脚本：

```bash
nano scripts/if-basic.sh
```

写入：

```bash
#!/usr/bin/env bash

name="$1"

if [ "$name" = "Alice" ]; then
  echo "Hello, Alice"
else
  echo "Hello, guest"
fi
```

运行：

```bash
bash scripts/if-basic.sh Alice
bash scripts/if-basic.sh Bob
```

注意：

```text
[ 和 ] 两边都需要空格。
```

---

## 2. 判断变量是否为空

`-z` 判断字符串长度是否为 0。

创建脚本：

```bash
nano scripts/check-name.sh
```

写入：

```bash
#!/usr/bin/env bash

name="$1"

if [ -z "$name" ]; then
  echo "Usage: $0 <name>"
  exit 1
fi

echo "Hello, $name"
```

运行：

```bash
bash scripts/check-name.sh
bash scripts/check-name.sh Alice
```

---

## 3. 文件和目录判断

常见判断：

| 写法 | 含义 |
| --- | --- |
| `[ -e "$path" ]` | 路径存在 |
| `[ -f "$file" ]` | 是普通文件 |
| `[ -d "$dir" ]` | 是目录 |
| `[ -r "$file" ]` | 可读 |
| `[ -w "$file" ]` | 可写 |
| `[ -x "$file" ]` | 可执行 |

创建脚本：

```bash
nano scripts/check-path.sh
```

写入：

```bash
#!/usr/bin/env bash

path="$1"

if [ -z "$path" ]; then
  echo "Usage: $0 <path>"
  exit 1
fi

if [ -f "$path" ]; then
  echo "File: $path"
elif [ -d "$path" ]; then
  echo "Directory: $path"
else
  echo "Not found or unsupported: $path"
  exit 1
fi
```

运行：

```bash
bash scripts/check-path.sh data/users.txt
bash scripts/check-path.sh logs
bash scripts/check-path.sh not-exist
```

---

## 4. 字符串判断

常见写法：

| 写法 | 含义 |
| --- | --- |
| `[ "$a" = "$b" ]` | 字符串相等 |
| `[ "$a" != "$b" ]` | 字符串不相等 |
| `[ -z "$a" ]` | 字符串为空 |
| `[ -n "$a" ]` | 字符串非空 |

示例：

```bash
role="$1"

if [ "$role" = "admin" ]; then
  echo "Admin user"
else
  echo "Normal user"
fi
```

---

## 5. 数字判断

数字判断使用专门操作符：

| 写法 | 含义 |
| --- | --- |
| `[ "$a" -eq "$b" ]` | 等于 |
| `[ "$a" -ne "$b" ]` | 不等于 |
| `[ "$a" -gt "$b" ]` | 大于 |
| `[ "$a" -ge "$b" ]` | 大于等于 |
| `[ "$a" -lt "$b" ]` | 小于 |
| `[ "$a" -le "$b" ]` | 小于等于 |

创建脚本：

```bash
nano scripts/check-number.sh
```

写入：

```bash
#!/usr/bin/env bash

num="$1"

if [ -z "$num" ]; then
  echo "Usage: $0 <number>"
  exit 1
fi

if [ "$num" -gt 10 ]; then
  echo "$num is greater than 10"
else
  echo "$num is less than or equal to 10"
fi
```

运行：

```bash
bash scripts/check-number.sh 5
bash scripts/check-number.sh 20
```

---

## 6. [[ ]] 简介

Bash 中也常用：

```bash
[[ condition ]]
```

它比 `[ ]` 更适合 Bash 脚本，支持更方便的模式匹配。

示例：

```bash
file="$1"

if [[ "$file" == *.log ]]; then
  echo "Log file"
else
  echo "Not a log file"
fi
```

新手阶段可以先熟练 `[ ]`，再逐渐使用 `[[ ]]`。

---

## 7. 本节小结

必须记住：

```bash
if [ condition ]; then
  echo "yes"
else
  echo "no"
fi

[ -z "$var" ]
[ -f "$file" ]
[ -d "$dir" ]
[ "$a" = "$b" ]
[ "$n" -gt 10 ]
[[ "$file" == *.log ]]
```

完成后进入下一节：`05-loops.md`。

