# 06：scp 与 rsync 文件传输

本节目标：学会在本机和远程 Linux 之间复制文件、复制目录，并理解 `scp` 和 `rsync` 的使用场景。

本节命令顺序：

```text
1. scp local remote
2. scp remote local
3. scp -r
4. rsync -av
5. rsync -avz
6. rsync --delete
7. rsync --dry-run
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-06
```

---

## 1. 准备练习文件

创建文件：

```bash
echo "hello ssh transfer" > data/message.txt
mkdir -p data/project
echo "project readme" > data/project/README.md
echo "app config" > data/project/app.conf
```

查看：

```bash
tree data
```

---

## 2. scp：复制本地文件到远程

格式：

```bash
scp local-file user@host:/remote/path/
```

示例：

```bash
scp data/message.txt user@host:/tmp/
```

指定端口：

```bash
scp -P 2222 data/message.txt user@host:/tmp/
```

注意：

```text
ssh 使用 -p 指定端口，scp 使用 -P 指定端口。
```

---

## 3. scp：复制远程文件到本地

格式：

```bash
scp user@host:/remote/file local-dir/
```

示例：

```bash
scp user@host:/tmp/message.txt downloads/
```

---

## 4. scp -r：复制目录

复制本地目录到远程：

```bash
scp -r data/project user@host:/tmp/
```

复制远程目录到本地：

```bash
scp -r user@host:/tmp/project downloads/
```

`scp` 简单直接，适合临时复制少量文件。

---

## 5. rsync：同步文件和目录

`rsync` 比 `scp` 更适合同步目录。它会比较差异，只传变化的内容。

同步目录到远程：

```bash
rsync -av data/project/ user@host:/tmp/project/
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-a` | 归档模式，保留权限、时间等信息 |
| `-v` | 显示详细过程 |
| `-z` | 压缩传输 |

加压缩：

```bash
rsync -avz data/project/ user@host:/tmp/project/
```

---

## 6. rsync 目录斜杠区别

注意这两个命令不同：

```bash
rsync -av data/project user@host:/tmp/
```

结果：

```text
/tmp/project/...
```

而：

```bash
rsync -av data/project/ user@host:/tmp/project/
```

结果：

```text
把 project 目录里面的内容同步到 /tmp/project/
```

简单记法：

```text
源目录后面有 /，表示同步目录里面的内容。
源目录后面没有 /，表示同步这个目录本身。
```

---

## 7. rsync --delete：让目标和源保持一致

如果源目录删除了文件，目标目录默认不会自动删除。

使用：

```bash
rsync -av --delete data/project/ user@host:/tmp/project/
```

危险提醒：

```text
--delete 会删除目标端多余文件，使用前一定确认源和目标路径。
```

---

## 8. rsync --dry-run：预演

预演，不真正执行：

```bash
rsync -av --dry-run data/project/ user@host:/tmp/project/
```

带删除的预演：

```bash
rsync -av --delete --dry-run data/project/ user@host:/tmp/project/
```

正式执行前，尤其是带 `--delete` 时，建议先 `--dry-run`。

---

## 9. 本节小结

必须记住：

```bash
scp file user@host:/tmp/
scp user@host:/tmp/file downloads/
scp -P 2222 file user@host:/tmp/
scp -r dir user@host:/tmp/
rsync -av dir/ user@host:/tmp/dir/
rsync -avz dir/ user@host:/tmp/dir/
rsync -av --delete --dry-run dir/ user@host:/tmp/dir/
```

完成后进入下一节：`07-firewall-ufw.md`。

