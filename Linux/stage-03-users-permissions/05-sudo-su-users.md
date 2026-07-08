# 05：sudo、su 与用户管理

本节目标：理解 `sudo` 和 `su` 的区别，学会创建练习用户、练习组，并管理用户所属组。

本节命令顺序：

```text
1. sudo
2. su
3. groupadd
4. useradd
5. passwd
6. usermod
7. id username
```

本节包含需要管理员权限的命令。请在 WSL、虚拟机或测试环境中练习，不要在重要服务器上随意创建用户。

---

## 1. sudo 是什么

`sudo` 表示以管理员权限执行一条命令。

示例：

```bash
sudo apt update
```

第一次使用 `sudo` 时，会要求输入当前用户密码。

注意：

```text
输入密码时终端不显示星号，这是正常现象。
```

查看自己是否能使用 `sudo`：

```bash
groups
```

如果输出里有 `sudo`，通常说明可以使用 `sudo`。

---

## 2. sudo 的安全习惯

执行任何 `sudo` 命令前，先问自己三件事：

```text
1. 这条命令会改什么？
2. 作用范围是不是只在我的练习目录？
3. 如果写错了，能不能恢复？
```

危险示例，不要执行：

```bash
sudo chmod -R 777 /
sudo chown -R user:user /
sudo rm -rf /
```

---

## 3. su 是什么

`su` 用来切换用户。

切换到 root：

```bash
su -
```

很多 Ubuntu 环境默认不允许直接用 root 密码登录，所以这个命令可能失败。

切换到某个普通用户：

```bash
su - username
```

退出当前切换的用户：

```bash
exit
```

新手阶段更常用的是 `sudo command`，而不是长期进入 root 用户。

---

## 4. groupadd：创建用户组

创建练习组：

```bash
sudo groupadd linuxstudy
```

查看是否创建成功：

```bash
getent group linuxstudy
```

如果提示组已存在，可以换一个名字，例如：

```bash
sudo groupadd linuxstudy2
```

---

## 5. useradd：创建用户

创建练习用户：

```bash
sudo useradd -m -s /bin/bash student1
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-m` | 创建家目录 |
| `-s /bin/bash` | 指定登录 Shell |

查看用户：

```bash
id student1
ls -ld /home/student1
```

---

## 6. passwd：设置密码

给练习用户设置密码：

```bash
sudo passwd student1
```

根据提示输入两次密码。

如果只是学习，不要设置太简单的常用密码；如果在公网服务器上，更要谨慎。

---

## 7. usermod：修改用户所属组

把 `student1` 加入 `linuxstudy` 组：

```bash
sudo usermod -aG linuxstudy student1
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `-a` | append，追加 |
| `-G` | 指定附加组 |

一定要一起使用 `-aG`。如果只用 `-G`，可能覆盖原有附加组。

查看结果：

```bash
id student1
groups student1
```

---

## 8. 切换到练习用户

切换：

```bash
su - student1
```

查看身份：

```bash
whoami
id
pwd
```

退出：

```bash
exit
```

---

## 9. 删除练习用户和组

如果你想清理练习用户：

```bash
sudo userdel -r student1
```

删除练习组：

```bash
sudo groupdel linuxstudy
```

注意：

```text
只删除自己创建的练习用户和练习组。
```

---

## 10. 本节小结

必须记住：

```bash
sudo command
su - username
exit
sudo groupadd groupname
sudo useradd -m -s /bin/bash username
sudo passwd username
sudo usermod -aG groupname username
id username
groups username
```

完成后进入下一节：`06-umask-special-permissions.md`。

