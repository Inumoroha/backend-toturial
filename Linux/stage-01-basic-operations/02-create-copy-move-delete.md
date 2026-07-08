# 02：创建、复制、移动与删除

本节目标：学会管理文件和目录，包括创建、复制、移动、重命名和删除。

本节命令顺序：

```text
1. mkdir
2. rmdir
3. touch
4. cp
5. mv
6. rm
```

请确认你在练习目录中：

```bash
cd ~/linux-practice/stage-01
pwd
```

---

## 1. mkdir：创建目录

创建一个目录：

```bash
mkdir projects
```

查看结果：

```bash
ls -lah
```

一次创建多层目录：

```bash
mkdir -p projects/app/src
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-p` | 父目录不存在时一起创建；目录已存在时不报错 |

### 练习

执行：

```bash
mkdir -p workspace/{docs,logs,scripts,backup}
tree workspace
```

你应该看到：

```text
workspace
├── backup
├── docs
├── logs
└── scripts
```

---

## 2. rmdir：删除空目录

`rmdir` 只能删除空目录。

创建一个空目录：

```bash
mkdir empty-dir
```

删除它：

```bash
rmdir empty-dir
```

如果目录里有文件，`rmdir` 会失败。这是一个比较安全的删除命令。

### 练习

执行：

```bash
mkdir test-rmdir
rmdir test-rmdir
ls -lah
```

---

## 3. touch：创建空文件或更新时间

创建一个空文件：

```bash
touch notes.txt
```

查看：

```bash
ls -lah
```

一次创建多个文件：

```bash
touch a.txt b.txt c.txt
```

如果文件已经存在，`touch` 不会清空文件，只会更新文件的修改时间。

### 练习

执行：

```bash
cd ~/linux-practice/stage-01/workspace/docs
touch day01.md day02.md commands.md
ls -lah
```

---

## 4. cp：复制文件和目录

复制文件：

```bash
cp day01.md day01-copy.md
```

复制到另一个目录：

```bash
cp commands.md ../backup/
```

复制目录需要加 `-r`：

```bash
cd ~/linux-practice/stage-01
cp -r workspace workspace-copy
```

常用参数：

| 参数 | 含义 |
| --- | --- |
| `-r` | 递归复制目录 |
| `-i` | 覆盖前询问 |
| `-v` | 显示复制过程 |

更适合新手的复制方式：

```bash
cp -iv source.txt target.txt
```

### 练习

执行：

```bash
cd ~/linux-practice/stage-01/workspace/docs
cp -iv day01.md day01-backup.md
cp -iv day02.md ../backup/
ls -lah
ls -lah ../backup
```

---

## 5. mv：移动和重命名

`mv` 有两个常见用途：移动文件、重命名文件。

重命名文件：

```bash
mv old-name.txt new-name.txt
```

移动文件：

```bash
mv file.txt target-dir/
```

示例：

```bash
cd ~/linux-practice/stage-01/workspace/docs
mv day01-backup.md day01-renamed.md
mv day01-renamed.md ../backup/
```

常用参数：

| 参数 | 含义 |
| --- | --- |
| `-i` | 覆盖前询问 |
| `-v` | 显示移动过程 |

更适合新手的移动方式：

```bash
mv -iv source.txt target.txt
```

### 练习

执行：

```bash
cd ~/linux-practice/stage-01/workspace/docs
touch temp-name.txt
mv -iv temp-name.txt better-name.txt
mv -iv better-name.txt ../backup/
ls -lah
ls -lah ../backup
```

---

## 6. rm：删除文件和目录

`rm` 用来删除文件。这个命令要非常小心，因为 Linux 默认没有回收站。

删除文件：

```bash
rm file.txt
```

删除前询问：

```bash
rm -i file.txt
```

删除目录：

```bash
rm -r dir-name
```

删除目录并显示过程：

```bash
rm -rv dir-name
```

常用参数：

| 参数 | 含义 |
| --- | --- |
| `-i` | 删除前询问 |
| `-r` | 递归删除目录 |
| `-v` | 显示删除过程 |
| `-f` | 强制删除，不询问 |

新手阶段建议少用 `-f`。

### 安全删除习惯

删除前先确认自己在哪里：

```bash
pwd
```

再确认要删什么：

```bash
ls -lah
```

最后使用交互式删除：

```bash
rm -i target-file
```

### 练习

只在练习目录中执行：

```bash
cd ~/linux-practice/stage-01
touch delete-me.txt
ls -lah delete-me.txt
rm -i delete-me.txt
ls -lah
```

删除练习目录副本：

```bash
cd ~/linux-practice/stage-01
ls -lah workspace-copy
rm -rv workspace-copy
```

---

## 7. 本节小结

本节必须记住：

```bash
mkdir -p dir/subdir
rmdir empty-dir
touch file.txt
cp -iv source target
cp -r source-dir target-dir
mv -iv old new
rm -i file
rm -rv dir
```

最重要的安全习惯：

```bash
pwd
ls -lah
rm -i 文件名
```

完成后进入下一节：`03-view-files.md`。

