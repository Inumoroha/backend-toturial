# 07：ncdu 与安全清理

本节目标：学会使用 `ncdu` 交互式分析目录空间，并建立安全清理文件的流程。

本节命令顺序：

```text
1. ncdu
2. ncdu path
3. du -ah | sort -rh | head
4. find large files
5. rm -i
6. rm -r
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-07
```

---

## 1. 安装 ncdu

安装：

```bash
sudo apt update
sudo apt install ncdu
```

运行：

```bash
ncdu .
```

`ncdu` 会扫描当前目录，并用交互方式显示空间占用。

---

## 2. ncdu 常用按键

| 按键 | 作用 |
| --- | --- |
| 方向键 | 移动 |
| `Enter` | 进入目录 |
| `Backspace` | 返回上一级 |
| `d` | 删除选中项 |
| `q` | 退出 |
| `?` | 帮助 |

新手阶段建议先只用它查看，不要急着在 `ncdu` 里删除。

---

## 3. 用命令行复核

用 `ncdu` 找到大文件后，再用命令复核：

```bash
du -ah . | sort -rh | head -n 10
```

查看文件详情：

```bash
ls -lh path/to/file
file path/to/file
```

确认是不是可以删除。

---

## 4. 使用 find 查找大文件

查找大于 10M 的文件：

```bash
find . -type f -size +10M -exec ls -lh {} \;
```

查找大于 100M 的文件：

```bash
find . -type f -size +100M -exec ls -lh {} \;
```

按修改时间查找最近 7 天修改过的大文件：

```bash
find . -type f -size +10M -mtime -7 -exec ls -lh {} \;
```

---

## 5. 安全清理流程

推荐流程：

```text
1. df -h 确认哪个挂载点空间紧张
2. du 或 ncdu 定位大目录
3. ls/file/stat 确认文件用途
4. lsof 查看文件是否被进程占用
5. 先移动或备份，再删除
6. 删除后 df -h 验证空间是否释放
```

删除单个文件前：

```bash
ls -lh target-file
rm -i target-file
```

删除目录前：

```bash
du -sh target-dir
ls -lah target-dir
rm -r target-dir
```

---

## 6. 不建议新手清理的目录

不要在不了解用途时清理：

```text
/bin
/sbin
/usr
/lib
/lib64
/etc
/boot
/var/lib
```

可以重点检查但要谨慎处理：

```text
/var/log
/tmp
/var/tmp
用户家目录下的下载、缓存、构建产物
```

---

## 7. 本节练习

执行：

```bash
cd ~/linux-practice/stage-07
ncdu .
du -ah . | sort -rh | head -n 10
find . -type f -size +10M -exec ls -lh {} \;
lsof data/big-20m.bin
```

创建一个可删除练习文件：

```bash
truncate -s 15M tmp/delete-me.bin
ls -lh tmp/delete-me.bin
rm -i tmp/delete-me.bin
df -h .
```

---

## 8. 本节小结

必须记住：

```bash
ncdu .
du -ah . | sort -rh | head -n 10
find . -type f -size +10M -exec ls -lh {} \;
lsof file
rm -i file
df -h .
```

完成后进入下一节：`08-stage-practice.md`。

