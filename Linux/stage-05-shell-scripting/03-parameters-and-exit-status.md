# 03：脚本参数与退出状态

本节目标：学会接收脚本参数、遍历参数，并理解命令成功失败如何用退出状态表示。

本节语法/命令顺序：

```text
1. $0
2. $1 $2
3. $#
4. "$@"
5. $?
6. exit 0
7. exit 1
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-05
```

---

## 1. $0、$1、$2 是什么

创建脚本：

```bash
nano scripts/args.sh
```

写入：

```bash
#!/usr/bin/env bash

echo "Script name: $0"
echo "First arg: $1"
echo "Second arg: $2"
```

运行：

```bash
bash scripts/args.sh Alice developer
```

你会看到：

```text
Script name: scripts/args.sh
First arg: Alice
Second arg: developer
```

---

## 2. $#：参数数量

修改 `scripts/args.sh`：

```bash
#!/usr/bin/env bash

echo "Script name: $0"
echo "Arg count: $#"
echo "First arg: $1"
echo "Second arg: $2"
```

运行：

```bash
bash scripts/args.sh Alice developer Beijing
```

`$#` 表示参数数量。

---

## 3. "$@"：所有参数

创建脚本：

```bash
nano scripts/all-args.sh
```

写入：

```bash
#!/usr/bin/env bash

echo "Arg count: $#"

for arg in "$@"; do
  echo "Arg: $arg"
done
```

运行：

```bash
bash scripts/all-args.sh Alice "Bob Smith" Carol
```

推荐遍历参数时使用：

```bash
for arg in "$@"; do
  echo "$arg"
done
```

这样包含空格的参数不会被拆开。

---

## 4. $?：上一条命令的退出状态

Linux 命令执行后会返回退出状态：

```text
0 表示成功
非 0 表示失败
```

执行成功命令：

```bash
ls data/users.txt
echo "$?"
```

执行失败命令：

```bash
ls not-exist
echo "$?"
```

注意：`$?` 只保存上一条命令的状态，所以要立刻查看。

---

## 5. exit：主动结束脚本

创建脚本：

```bash
nano scripts/need-file.sh
```

写入：

```bash
#!/usr/bin/env bash

file="$1"

if [ -z "$file" ]; then
  echo "Usage: $0 <file>"
  exit 1
fi

echo "File argument: $file"
exit 0
```

运行：

```bash
bash scripts/need-file.sh
echo "$?"
```

再运行：

```bash
bash scripts/need-file.sh data/users.txt
echo "$?"
```

---

## 6. 常见参数检查模板

检查是否传入参数：

```bash
if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <file>"
  exit 1
fi
```

检查文件是否存在：

```bash
if [ ! -f "$1" ]; then
  echo "Error: file not found: $1"
  exit 1
fi
```

完整示例：

```bash
#!/usr/bin/env bash

if [ "$#" -lt 1 ]; then
  echo "Usage: $0 <file>"
  exit 1
fi

file="$1"

if [ ! -f "$file" ]; then
  echo "Error: file not found: $file"
  exit 1
fi

wc -l "$file"
```

---

## 7. 本节小结

必须记住：

```bash
echo "$0"
echo "$1"
echo "$2"
echo "$#"
for arg in "$@"; do
  echo "$arg"
done
echo "$?"
exit 0
exit 1
```

核心概念：

```text
脚本参数让脚本变得通用。
退出状态让脚本能表达成功或失败。
```

完成后进入下一节：`04-conditionals.md`。

