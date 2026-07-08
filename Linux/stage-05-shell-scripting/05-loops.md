# 05：循环

本节目标：学会使用 `for` 和 `while` 处理列表、文件和重复任务。

本节语法/命令顺序：

```text
1. for item in list
2. for file in *.log
3. while condition
4. while read -r line
5. break
6. continue
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-05
```

---

## 1. for 循环：遍历列表

创建脚本：

```bash
nano scripts/for-list.sh
```

写入：

```bash
#!/usr/bin/env bash

for name in Alice Bob Carol; do
  echo "Hello, $name"
done
```

运行：

```bash
bash scripts/for-list.sh
```

---

## 2. for 循环：遍历文件

创建几个日志文件：

```bash
touch logs/app.log logs/error.log logs/debug.log
```

创建脚本：

```bash
nano scripts/for-files.sh
```

写入：

```bash
#!/usr/bin/env bash

for file in logs/*.log; do
  echo "File: $file"
  wc -l "$file"
done
```

运行：

```bash
bash scripts/for-files.sh
```

注意变量加双引号：

```bash
"$file"
```

这样文件名包含空格时更安全。

---

## 3. while 循环：按条件重复

创建脚本：

```bash
nano scripts/while-count.sh
```

写入：

```bash
#!/usr/bin/env bash

count=1

while [ "$count" -le 5 ]; do
  echo "Count: $count"
  count=$((count + 1))
done
```

运行：

```bash
bash scripts/while-count.sh
```

`$(( ))` 用来做整数运算。

---

## 4. while read：逐行读取文件

创建脚本：

```bash
nano scripts/read-users.sh
```

写入：

```bash
#!/usr/bin/env bash

file="data/users.txt"

while read -r user; do
  echo "User: $user"
done < "$file"
```

运行：

```bash
bash scripts/read-users.sh
```

推荐逐行读取文件时使用：

```bash
while read -r line; do
  echo "$line"
done < "$file"
```

`-r` 可以避免反斜杠被特殊处理。

---

## 5. break：提前结束循环

创建脚本：

```bash
nano scripts/break-demo.sh
```

写入：

```bash
#!/usr/bin/env bash

for num in 1 2 3 4 5; do
  if [ "$num" -eq 3 ]; then
    echo "Stop at $num"
    break
  fi
  echo "$num"
done
```

运行：

```bash
bash scripts/break-demo.sh
```

---

## 6. continue：跳过本次循环

创建脚本：

```bash
nano scripts/continue-demo.sh
```

写入：

```bash
#!/usr/bin/env bash

for num in 1 2 3 4 5; do
  if [ "$num" -eq 3 ]; then
    echo "Skip $num"
    continue
  fi
  echo "$num"
done
```

运行：

```bash
bash scripts/continue-demo.sh
```

---

## 7. 循环处理脚本参数

创建脚本：

```bash
nano scripts/loop-args.sh
```

写入：

```bash
#!/usr/bin/env bash

if [ "$#" -eq 0 ]; then
  echo "Usage: $0 <arg1> [arg2...]"
  exit 1
fi

for arg in "$@"; do
  echo "Arg: $arg"
done
```

运行：

```bash
bash scripts/loop-args.sh one two "three words"
```

---

## 8. 本节小结

必须记住：

```bash
for item in list; do
  echo "$item"
done

for file in logs/*.log; do
  wc -l "$file"
done

while [ "$count" -le 5 ]; do
  count=$((count + 1))
done

while read -r line; do
  echo "$line"
done < "$file"

break
continue
```

完成后进入下一节：`06-functions-debugging.md`。

