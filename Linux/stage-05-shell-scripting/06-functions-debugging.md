# 06：函数、调试与脚本安全选项

本节目标：学会把重复逻辑封装成函数，并掌握基础脚本检查和调试方法。

本节语法/命令顺序：

```text
1. function_name() { ... }
2. local variable
3. return
4. bash -n script.sh
5. bash -x script.sh
6. set -e
7. set -u
8. set -o pipefail
9. set -euo pipefail
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-05
```

---

## 1. 定义函数

创建脚本：

```bash
nano scripts/functions-basic.sh
```

写入：

```bash
#!/usr/bin/env bash

say_hello() {
  echo "Hello, $1"
}

say_hello "Alice"
say_hello "Bob"
```

运行：

```bash
bash scripts/functions-basic.sh
```

函数适合封装重复逻辑。

---

## 2. local：函数内部变量

创建脚本：

```bash
nano scripts/functions-local.sh
```

写入：

```bash
#!/usr/bin/env bash

print_user() {
  local name="$1"
  local role="$2"
  echo "name=$name role=$role"
}

print_user "Alice" "admin"
print_user "Bob" "developer"
```

运行：

```bash
bash scripts/functions-local.sh
```

函数里的临时变量建议用 `local`。

---

## 3. 函数返回状态

Bash 函数一般通过退出状态表示成功失败。

创建脚本：

```bash
nano scripts/functions-return.sh
```

写入：

```bash
#!/usr/bin/env bash

file_exists() {
  local file="$1"

  if [ -f "$file" ]; then
    return 0
  else
    return 1
  fi
}

if file_exists "data/users.txt"; then
  echo "File exists"
else
  echo "File not found"
fi
```

运行：

```bash
bash scripts/functions-return.sh
```

---

## 4. bash -n：检查语法

`bash -n` 只检查语法，不执行脚本。

执行：

```bash
bash -n scripts/functions-basic.sh
```

如果没有输出，通常表示语法没问题。

故意创建一个错误脚本：

```bash
cat > scripts/syntax-error.sh <<'EOF'
#!/usr/bin/env bash

if [ -f data/users.txt ]; then
  echo "ok"
EOF
```

检查：

```bash
bash -n scripts/syntax-error.sh
```

你会看到缺少 `fi` 的错误。

---

## 5. bash -x：跟踪执行过程

`bash -x` 会显示脚本每一步执行了什么。

执行：

```bash
bash -x scripts/functions-local.sh
```

调试变量、条件判断、循环时非常有用。

也可以在脚本中局部开启：

```bash
set -x
echo "debug this"
set +x
```

---

## 6. set -e：出错就退出

`set -e` 表示命令失败时退出脚本。

示例：

```bash
#!/usr/bin/env bash
set -e

mkdir -p tmp/demo
cp not-exist.txt tmp/demo/
echo "This line will not run if cp fails"
```

它能避免脚本在前一步失败后继续执行。

---

## 7. set -u：使用未定义变量时报错

`set -u` 可以帮助发现变量名写错。

示例：

```bash
#!/usr/bin/env bash
set -u

echo "$not_defined_variable"
```

运行时会报错。

---

## 8. set -o pipefail：管道失败也算失败

默认情况下，管道命令的退出状态通常取最后一个命令。

使用：

```bash
set -o pipefail
```

可以让管道中任何一步失败都被识别。

常见组合：

```bash
set -euo pipefail
```

建议在稍正式的脚本开头使用：

```bash
#!/usr/bin/env bash
set -euo pipefail
```

新手注意：加了这些选项后，脚本会更严格，报错也更早，需要认真处理变量和失败情况。

---

## 9. 本节小结

必须记住：

```bash
my_func() {
  local value="$1"
  echo "$value"
}

return 0
return 1
bash -n script.sh
bash -x script.sh
set -euo pipefail
```

完成后进入下一节：`07-practical-scripts.md`。

