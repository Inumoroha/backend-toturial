# 03：find 与 xargs

本节目标：学会按文件名、类型、大小、时间查找文件，并用 `xargs` 批量处理结果。

本节命令顺序：

```text
1. find . -name
2. find . -type
3. find . -size
4. find . -mtime
5. find . -exec
6. xargs
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-02
```

---

## 1. 准备更多文件

执行：

```bash
mkdir -p tmp/find-demo/{a,b,c}
touch tmp/find-demo/a/app.log
touch tmp/find-demo/a/error.log
touch tmp/find-demo/b/readme.md
touch tmp/find-demo/b/config.conf
touch tmp/find-demo/c/notes.txt
echo "ERROR database timeout" > tmp/find-demo/a/error.log
echo "server_port=8080" > tmp/find-demo/b/config.conf
```

查看结构：

```bash
tree tmp/find-demo
```

---

## 2. 按文件名查找：`-name`

查找所有 `.log` 文件：

```bash
find tmp/find-demo -name "*.log"
```

查找名为 `config.conf` 的文件：

```bash
find tmp/find-demo -name "config.conf"
```

忽略大小写查找：

```bash
find tmp/find-demo -iname "*.LOG"
```

---

## 3. 按类型查找：`-type`

查找普通文件：

```bash
find tmp/find-demo -type f
```

查找目录：

```bash
find tmp/find-demo -type d
```

常见类型：

| 类型 | 含义 |
| --- | --- |
| `f` | 普通文件 |
| `d` | 目录 |
| `l` | 符号链接 |

---

## 4. 按大小查找：`-size`

查找大于 0 字节的文件：

```bash
find tmp/find-demo -type f -size +0c
```

查找小于 1 KB 的文件：

```bash
find tmp/find-demo -type f -size -1k
```

常见单位：

| 单位 | 含义 |
| --- | --- |
| `c` | 字节 |
| `k` | KB |
| `M` | MB |
| `G` | GB |

---

## 5. 按修改时间查找：`-mtime`

查找 1 天内修改过的文件：

```bash
find tmp/find-demo -type f -mtime -1
```

查找 7 天以前修改过的文件：

```bash
find tmp/find-demo -type f -mtime +7
```

含义：

| 写法 | 含义 |
| --- | --- |
| `-mtime -1` | 1 天内修改过 |
| `-mtime 1` | 大约 1 天前修改 |
| `-mtime +7` | 7 天以前修改 |

---

## 6. `-exec`：对查找结果执行命令

查看所有 `.log` 文件内容：

```bash
find tmp/find-demo -name "*.log" -exec cat {} \;
```

显示所有 `.log` 文件详细信息：

```bash
find tmp/find-demo -name "*.log" -exec ls -lah {} \;
```

解释：

| 部分 | 含义 |
| --- | --- |
| `{}` | 当前找到的文件 |
| `\;` | `-exec` 命令结束标记 |

---

## 7. xargs：把输入变成命令参数

`xargs` 常和管道一起使用。

查找 `.log` 文件并显示内容：

```bash
find tmp/find-demo -name "*.log" | xargs cat
```

查找 `.log` 文件并统计行数：

```bash
find tmp/find-demo -name "*.log" | xargs wc -l
```

### 文件名安全问题

如果文件名里有空格，普通 `xargs` 可能出错。更安全的写法：

```bash
find tmp/find-demo -name "*.log" -print0 | xargs -0 cat
```

初学阶段先掌握普通写法，知道有空格时用 `-print0` 和 `xargs -0`。

---

## 8. find + grep 组合

在所有 `.log` 文件中搜索 `ERROR`：

```bash
find tmp/find-demo -name "*.log" | xargs grep "ERROR"
```

更安全写法：

```bash
find tmp/find-demo -name "*.log" -print0 | xargs -0 grep "ERROR"
```

---

## 9. 本节小结

必须记住：

```bash
find . -name "*.log"
find . -iname "*.log"
find . -type f
find . -type d
find . -type f -size +10M
find . -type f -mtime -7
find . -name "*.log" -exec ls -lah {} \;
find . -name "*.log" | xargs grep "ERROR"
```

完成后进入下一节：`04-count-sort-uniq.md`。

