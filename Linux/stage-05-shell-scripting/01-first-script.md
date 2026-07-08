# 01：第一个 Shell 脚本

本节目标：认识 Shell 脚本文件，学会编写、执行和授权一个最小脚本。

本节语法/命令顺序：

```text
1. #!/usr/bin/env bash
2. echo
3. bash script.sh
4. chmod +x script.sh
5. ./script.sh
6. ls -l script.sh
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-05
```

---

## 1. Shell 脚本是什么

Shell 脚本就是把多条命令写进一个文件，然后让 Shell 按顺序执行。

你平时手动执行：

```bash
pwd
date
whoami
```

写成脚本后，就可以一次运行。

---

## 2. 创建第一个脚本

创建脚本文件：

```bash
nano scripts/hello.sh
```

如果没有 `nano`，可以安装：

```bash
sudo apt install nano
```

在文件中写入：

```bash
#!/usr/bin/env bash

echo "Hello, Linux"
echo "Today is:"
date
echo "Current user:"
whoami
echo "Current directory:"
pwd
```

保存并退出：

```text
Ctrl + O
Enter
Ctrl + X
```

如果你使用 VS Code，也可以直接编辑这个文件。

---

## 3. Shebang 是什么

第一行：

```bash
#!/usr/bin/env bash
```

叫 Shebang。它告诉系统：

```text
请用 bash 来执行这个脚本。
```

推荐使用：

```bash
#!/usr/bin/env bash
```

而不是固定写死：

```bash
#!/bin/bash
```

因为 `/usr/bin/env bash` 更容易适配不同系统。

---

## 4. 用 bash 执行脚本

执行：

```bash
bash scripts/hello.sh
```

这种方式不要求脚本有执行权限，因为是你主动调用 `bash` 来读取脚本。

---

## 5. 添加执行权限

查看权限：

```bash
ls -l scripts/hello.sh
```

添加执行权限：

```bash
chmod +x scripts/hello.sh
```

再次查看：

```bash
ls -l scripts/hello.sh
```

你应该能看到权限中出现 `x`。

---

## 6. 直接执行脚本

执行：

```bash
./scripts/hello.sh
```

这里的 `./` 表示当前目录下的路径。

如果你只写：

```bash
scripts/hello.sh
```

在当前目录下也能执行，因为 `scripts/hello.sh` 是相对路径。

如果只写：

```bash
hello.sh
```

通常会失败，因为当前目录不在系统命令搜索路径里。

---

## 7. 两种执行方式的区别

| 执行方式 | 是否需要执行权限 | 是否读取 Shebang |
| --- | --- | --- |
| `bash script.sh` | 不需要 | 不依赖 |
| `./script.sh` | 需要 | 依赖 |

实际写脚本时，两种方式都常见。

---

## 8. 本节练习

创建脚本：

```bash
nano scripts/system-info.sh
```

写入：

```bash
#!/usr/bin/env bash

echo "User: $(whoami)"
echo "Host: $(hostname)"
echo "Date: $(date '+%F %T')"
echo "Kernel:"
uname -a
```

运行：

```bash
bash scripts/system-info.sh
chmod +x scripts/system-info.sh
./scripts/system-info.sh
```

---

## 9. 本节小结

必须记住：

```bash
#!/usr/bin/env bash
echo "message"
bash scripts/hello.sh
chmod +x scripts/hello.sh
./scripts/hello.sh
ls -l scripts/hello.sh
```

完成后进入下一节：`02-variables-and-quotes.md`。

