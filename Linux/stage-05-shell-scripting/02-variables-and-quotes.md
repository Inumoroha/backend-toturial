# 02：变量与引用

本节目标：学会定义变量、引用变量，并理解双引号、单引号和命令替换。

本节语法/命令顺序：

```text
1. name=value
2. echo "$name"
3. echo "${name}"
4. "double quotes"
5. 'single quotes'
6. command=$(command)
7. readonly
8. unset
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-05
```

---

## 1. 定义变量

Shell 变量写法：

```bash
name="Alice"
```

注意等号两边不能有空格。

正确：

```bash
name="Alice"
```

错误：

```bash
name = "Alice"
```

---

## 2. 引用变量

创建脚本：

```bash
nano scripts/variables.sh
```

写入：

```bash
#!/usr/bin/env bash

name="Alice"
role="developer"

echo "$name"
echo "$role"
echo "User $name is a $role"
echo "User ${name} is a ${role}"
```

运行：

```bash
bash scripts/variables.sh
```

推荐使用：

```bash
"$name"
```

或者：

```bash
"${name}"
```

变量和普通字符连在一起时，`${name}` 更清楚。

---

## 3. 双引号

双引号中会展开变量：

```bash
name="Alice"
echo "Hello $name"
```

输出：

```text
Hello Alice
```

双引号还能保留空格：

```bash
file="my note.txt"
echo "$file"
```

脚本里引用变量时，默认优先加双引号。

---

## 4. 单引号

单引号中不会展开变量：

```bash
name="Alice"
echo 'Hello $name'
```

输出：

```text
Hello $name
```

单引号适合输出原样文本。

---

## 5. 为什么变量要加双引号

创建脚本：

```bash
nano scripts/quote-demo.sh
```

写入：

```bash
#!/usr/bin/env bash

file="my note.txt"

touch "$file"

echo "With quotes:"
ls -l "$file"

echo "Without quotes:"
ls -l $file
```

运行：

```bash
bash scripts/quote-demo.sh
```

没有双引号时，`my note.txt` 可能被拆成三个参数：

```text
my
note.txt
```

所以脚本里推荐写：

```bash
"$file"
```

---

## 6. 命令替换

命令替换可以把命令输出保存到变量：

```bash
today="$(date '+%F')"
```

示例脚本：

```bash
nano scripts/command-substitution.sh
```

写入：

```bash
#!/usr/bin/env bash

today="$(date '+%F')"
user="$(whoami)"
current_dir="$(pwd)"

echo "Today: $today"
echo "User: $user"
echo "Directory: $current_dir"
```

运行：

```bash
bash scripts/command-substitution.sh
```

推荐写法：

```bash
result="$(command)"
```

旧写法反引号也能用：

```bash
result=`command`
```

但新手阶段建议统一使用 `$()`。

---

## 7. readonly 和 unset

创建只读变量：

```bash
readonly app_name="linux-toolkit"
```

删除变量：

```bash
unset temp_value
```

练习：

```bash
temp_value="hello"
echo "$temp_value"
unset temp_value
echo "$temp_value"
```

`readonly` 用得不如普通变量多，先认识即可。

---

## 8. 本节小结

必须记住：

```bash
name="Alice"
echo "$name"
echo "${name}"
echo "Hello $name"
echo 'Hello $name'
today="$(date '+%F')"
unset name
```

核心习惯：

```text
变量赋值等号两边不要有空格。
引用变量时优先加双引号。
命令替换优先使用 $(command)。
```

完成后进入下一节：`03-parameters-and-exit-status.md`。

